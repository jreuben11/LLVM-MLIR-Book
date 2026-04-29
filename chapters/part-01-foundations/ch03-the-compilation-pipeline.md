# Chapter 3 — The Compilation Pipeline

*Part I — Foundations*

Every compiler claim reduces to a question about some pipeline stage: "does the optimiser see this value?" — ask at the IR level; "does the inliner fire here?" — ask at the CGSCC level; "is this addressed correctly?" — ask at MIR. The LLVM compilation pipeline is not a black box that turns source into binaries. It is a sequence of well-defined representation transformations, each observable and manipulable with the right tool invocation. This chapter maps the full pipeline from `clang driver` to linked binary, identifies the role of every tool in the ecosystem, shows how to observe every intermediate representation, and explains the design decisions that make each stage necessary.

---

## 3.1 The Compilation Pipeline at 10,000 Feet

The canonical C/C++ path through the LLVM toolchain traverses seven layers of representation. Two parallel paths exist for instruction selection: the classic SelectionDAG path and the newer GlobalISel path. The high-level picture:

```
  Source (.c / .cpp / .f90 / .mlir)
         │
         │  clang -cc1 (Preprocessor)
         ▼
  Preprocessed Source (.i / .ii)
         │
         │  clang -cc1 (Lexer → Parser → Sema)
         ▼
  Clang AST
         │
         │  clang -cc1 (CodeGen)           [optional: CIR via -fclangir]
         ▼
  LLVM IR (.ll text / .bc bitcode)
         │
         │  opt  (Middle-End Pass Pipeline)
         ▼
  Optimised LLVM IR
         │
         ├─────────────────────────────────────────┐
         │  llc (SelectionDAG ISel)                │  llc (GlobalISel)
         ▼                                         ▼
  SelectionDAG → MachineDAG              Generic MachineIR (gMIR)
         │                                         │
         │  llc (MachineIR passes)                 │  llc (Legalisation → RegAlloc)
         ▼                                         ▼
  MachineIR (.mir)  ◄───────────────────────────────
         │
         │  llc (MC layer: AsmPrinter / ObjectWriter)
         ▼
  MC (Assembly .s / Object file .o / .bc for LTO)
         │
         │  lld / ld  (Linker)
         ▼
  Linked Binary (ELF / Mach-O / PE-COFF)
```

The SelectionDAG path (the default for x86, AArch64, and most targets) builds a directed-acyclic graph per basic block, runs target-independent DAG combines, legalises types and operations into what the target supports, then does selection to target-specific MachineSDNode. GlobalISel (the default on AArch64 at `-O0` since LLVM 13, progressively adopted elsewhere) stays in a register-based form throughout: IR is lowered to generic MIR (`G_ADD`, `G_LOAD`, …), legalised in-place, then selected to real opcodes without ever building a DAG. The rest of the MIR pipeline — register allocation, prologue/epilogue insertion, branch relaxation — is shared by both paths. [Chapter 84 covers SelectionDAG in depth; Chapter 85 covers GlobalISel.]

The pipeline diagram above omits two important facts. First, in a normal `clang` invocation the middle-end (opt) and backend (llc) run inside the same process: the pass manager is invoked directly by cc1's `EmitBackendOutput` function, which calls `llvm/lib/CodeGen/BackendUtil.cpp`. The `opt` and `llc` tools are convenience frontends for driving the same pipelines from files; they are not separate processes in the default build flow. Second, the stage boundaries are soft when running cc1 directly: the IR is never written to disk between the middle-end and backend in a `-O2 -c` compilation unless you explicitly add `-save-temps`.

---

## 3.2 Tools and Their Roles

Understanding which tool owns which stage prevents the common error of applying a flag to the wrong program.

### The clang Driver

`clang` (the binary) is not a compiler. It is a **driver** — a thin coordinator that:

1. Parses the command line using `clang/lib/Driver/Driver.cpp`.
2. Determines the toolchain for the target triple.
3. Constructs a **job list** (`clang/lib/Driver/Compilation.cpp`): a directed graph of compilation, assemble, and link jobs.
4. Forks child processes (usually `clang -cc1` for compilation and a linker) to execute the jobs.

Critically, the driver translates high-level flags into lower-level cc1 flags. `-O2` becomes dozens of `-mllvm` and cc1 flags; `-fsanitize=address` triggers both cc1 instrumentation flags and linker input changes. The driver also handles multi-file compilation, LTO coordination, and offload bundling.

### cc1: The Real Compiler

`clang -cc1` is the actual compilation process. It runs the preprocessor, parser, semantic analysis, and code generator in a single process. When you invoke `clang foo.c`, the driver re-executes itself with the `-cc1` flag. On most systems:

```
$ /usr/lib/llvm-22/bin/clang foo.c
```

is exactly equivalent to a two-step job:

```
$ /usr/lib/llvm-22/bin/clang -cc1 [flags...] foo.c -emit-obj -o foo.o
$ /usr/bin/ld [flags...] foo.o -o a.out
```

The driver and cc1 are described in full in [Chapter 28 — The Clang Driver](../part-05-clang-frontend/ch28-the-clang-driver.md).

### opt

`opt` is the standalone IR-to-IR optimizer. It reads LLVM IR (textual `.ll` or bitcode `.bc`), applies a pass pipeline, and writes transformed IR. In LLVM 22 opt uses the new pass manager exclusively — the legacy pass manager (`-enable-new-pm=false`) is gone. Pass pipelines are specified with `-passes='...'`. It does not invoke any frontend or backend code.

### llc

`llc` is the standalone IR-to-machine-code compiler. It reads LLVM IR or bitcode and emits assembly, object code, or MIR (with `-stop-after=<pass>`). It is the tool that runs the instruction selector, register allocator, and MC layer.

### llvm-as and llvm-dis

`llvm-as` compiles textual IR (`.ll`) to bitcode (`.bc`). `llvm-dis` decompiles bitcode back to textual IR. They are lossless round-trip tools — useful for inspecting bitcode produced by LTO pipelines and for preprocessing IR files before analysis.

### lld

`lld` is the LLVM linker. It supports ELF (`ld.lld`), Mach-O (`ld64.lld`), PE-COFF (`lld-link`), and WebAssembly (`wasm-ld`). Unlike the GNU linker, lld is designed as a library: the same core logic runs all format variants. When LTO is active, lld drives the final IR optimization and code generation step by loading the bitcode from `.o` files and calling back into LLVM. The linker wrapper for GPU offload (`clang-linker-wrapper`) wraps the host linker and handles device-code extraction.

### llvm-link

`llvm-link` merges multiple LLVM IR modules into one. It is conceptually the IR-level equivalent of the system linker's `.o` merging step, but operates before code generation. The principal use cases are whole-program analysis, LTO experiments at the IR level, and linking together modules produced by different frontends.

### llvm-ar

`llvm-ar` creates and manipulates LLVM bitcode archives (`.a` files containing `.bc` members). It also handles native object archives; the format is autodetected. The thin-archive mode (`-T`) stores only the file paths rather than the member data, which dramatically reduces I/O for large projects.

### mlir-opt and mlir-translate

`mlir-opt` is the MLIR analog of `opt`. It reads MLIR text, applies a pass pipeline, and writes transformed MLIR. `mlir-translate` converts between MLIR and other representations — most importantly, it materialises LLVM IR from the LLVM dialect of MLIR (`--mlir-to-llvmir`). Additional MLIR tools in the toolchain include `mlir-reduce` (a test case reducer for MLIR bugs, analogous to `llvm-reduce`), `mlir-query` (structural query over MLIR ops), `mlir-runner` (JIT execution of MLIR modules via ORC), and `mlir-lsp-server` (Language Server Protocol server for `.mlir` files).

### lli

`lli` is the LLVM interpreter and JIT driver. It reads LLVM IR and either interprets it or JIT-compiles it using ORC (the default since LLVM 12). It is the canonical smoke-test tool for new IR and a thin wrapper around the ORC JIT APIs.

### llvm-mc

`llvm-mc` is the LLVM Machine Code tool. It assembles `.s` input to an object file (`--filetype=obj`), disassembles object files (`-disassemble`), and can pretty-print the instruction encoding (`-show-encoding`). It is the command-line interface to the MC layer and is invaluable when verifying encoding decisions for new target instructions or inline assembly constraints.

### Supporting Tools

Several additional tools appear throughout the LLVM ecosystem. `llvm-nm` lists symbols in object files and archives. `llvm-objcopy` and `llvm-strip` manipulate object file sections and debug info. `llvm-dwarfdump` inspects DWARF debug information in detail. `llvm-cov` reports test coverage from profile data. `llvm-profdata` merges and converts profile data files between formats. `llvm-mca` (Machine Code Analyser) takes assembly and produces a detailed throughput analysis based on the target's pipeline model — an excellent tool for micro-optimisation work.

---

## 3.3 The Driver and cc1: Peeling Back the Layers

The single most useful debugging technique for compiler development is revealing the exact cc1 invocation the driver constructs. The `-###` flag makes the driver print every job it would run — without running them:

```bash
$ /usr/lib/llvm-22/bin/clang -### /tmp/add.c 2>&1
```

For the canonical input `int add(int a, int b) { return a + b; }`, LLVM 22 produces two jobs. The first is the cc1 invocation:

```
"/usr/lib/llvm-22/bin/clang" "-cc1"
  # --- Target and triple ---
  "-triple" "x86_64-pc-linux-gnu"
  # --- Output mode ---
  "-emit-obj"
  # --- Codegen control ---
  "-mrelocation-model" "pic"  "-pic-level" "2"  "-pic-is-pie"
  "-mframe-pointer=all"
  "-fmath-errno"  "-ffp-contract=on"  "-fno-rounding-math"
  "-mconstructor-aliases"
  "-funwind-tables=2"
  "-target-cpu" "x86-64"  "-tune-cpu" "generic"
  # --- Debug info ---
  "-debugger-tuning=gdb"
  "-fdebug-compilation-dir=..."  "-fcoverage-compilation-dir=..."
  # --- Resource and header search ---
  "-resource-dir" "/usr/lib/llvm-22/lib/clang/22"
  "-internal-isystem" "/usr/lib/llvm-22/lib/clang/22/include"
  "-internal-isystem" "/usr/local/include"
  "-internal-externc-isystem" "/usr/include/x86_64-linux-gnu"
  "-internal-externc-isystem" "/usr/include"
  # --- Diagnostics ---
  "-ferror-limit" "19"
  "-fgnuc-version=4.2.1"
  # --- Misc ---
  "-faddrsig"  "-fdwarf2-cfi-asm"
  # --- Input/output ---
  "-o" "/tmp/add-XXXXXX.o"
  "-x" "c"  "/tmp/add.c"
```

The second job is the linker invocation, using `/usr/bin/ld` (the system linker — on this Ubuntu installation, lld 22 is not packaged as a separate binary, so the GNU linker is used by default; pass `-fuse-ld=lld` to request lld explicitly).

**Why reading cc1 matters for compiler development.** When you add a new cc1 flag in Clang, the driver may need to be updated to translate a driver-level flag to it. When a cc1 flag interaction produces wrong output, `-###` tells you exactly what was passed. When writing a Clang plugin or a libtooling tool that constructs its own `CompilerInvocation`, you need to know which cc1 flags correspond to which semantic behaviours. The mapping is in [`clang/lib/Driver/ToolChains/Clang.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/Clang.cpp) (the `Clang::ConstructJob` method) and is extensive.

A concrete example of the flag-translation gap: the driver flag `-O2` is not simply forwarded to cc1 as `-O2`. The driver also sets `-mframe-pointer=none` (suppresses frame pointers at -O1 and above), enables vectorisation (`-vectorize-loops`, `-vectorize-slp`), and may enable loop unrolling. The cc1 `-O2` flag then drives the pass manager's `default<O2>` pipeline. Passing `-cc1 -O2` directly without the other flags the driver adds would silently omit some optimisation side-effects. This is why targeting cc1 directly — as `clang -cc1 ...` — is fine for controlled experiments but not for production builds.

A second subtlety: the `-fgnuc-version=4.2.1` flag tells cc1 to pretend to be GCC 4.2.1 for the purposes of `__GNUC__` version macros. This compat flag is set even on Linux x86-64 by default, because many system headers have version guards that test `__GNUC__`. The driver injects it via `clang/lib/Driver/ToolChains/Clang.cpp:AddGNUCPlusPlusIncludePaths`.

**Key cc1 flag categories:**

| Category | Flag examples | Effect |
|---|---|---|
| Compilation mode | `-emit-obj`, `-emit-llvm`, `-emit-llvm-bc`, `-S`, `-E`, `-fsyntax-only` | What the compiler produces |
| Target | `-triple`, `-target-cpu`, `-target-feature` | Machine the code runs on |
| Relocation/PIC | `-mrelocation-model pic`, `-pic-level 2`, `-pic-is-pie` | PIC/PIE mode |
| Optimisation | `-O0`…`-O3`, `-Os`, `-Oz` | Opt level passed to the IR pipeline |
| Language | `-x c`, `-x c++`, `-std=c++23` | Input language |
| Header search | `-internal-isystem`, `-internal-externc-isystem`, `-I` | Where to find headers |
| Debug info | `-g`, `-debugger-tuning=gdb`, `-dwarf-version=5` | Debug metadata level |
| LTO | `-flto=thin`, `-flto-unit`, `-emit-llvm-bc` | Link-time optimisation mode |
| Sanitizers | `-fsanitize=address`, `-fsanitize-recover=all` | Runtime instrumentation |
| Profiling | `-fprofile-generate`, `-fprofile-use` | PGO instrumentation |
| EH/unwind | `-funwind-tables=2`, `-fexceptions`, `-fcxx-exceptions` | Exception support |

---

## 3.4 Observing Each Stage

The primary skill of a compiler engineer is knowing how to inspect what the pipeline produces at each stage. Every representation level is observable without recompiling LLVM.

### Stage 0: Preprocessed Source

```bash
$ /usr/lib/llvm-22/bin/clang -E /tmp/add.c
```

Output (abbreviated):

```c
# 1 "/tmp/add.c"
# 1 "<built-in>" 1
# 412 "<built-in>" 3
# 1 "<command line>" 1
int add(int a, int b) { return a + b; }
```

The `# N "file"` directives are linemarkers that preserve source location tracking through the subsequent parser. `-E` also runs macro expansion — critical for debugging header inclusion order and macro disasters. Use `-E -dD` to also dump macro definitions, or `-E -v` to trace header searches.

Additional preprocessor inspection commands:

```bash
# Show all macros that become defined (including built-ins):
$ clang -E -dM /tmp/add.c | head -20

# Show only the preprocessed source without linemarkers:
$ clang -E -P /tmp/add.c

# Trace header file inclusion (which file is included, from where):
$ clang -E -H /tmp/add.c 2>&1 | grep "^\\."

# Dependency generation (Makefile-style, for build systems):
$ clang -MM /tmp/add.c
```

The `-MM` flag generates Make-compatible dependency rules, which is the mechanism `make` and `ninja` use to perform incremental header-sensitive rebuilds. It invokes the preprocessor but suppresses output of the preprocessed source.

### Stage 1: The AST

```bash
$ /usr/lib/llvm-22/bin/clang -Xclang -ast-dump /tmp/add.c
```

(Note: `-ast-dump` is a cc1 flag, not a driver flag; it must be passed via `-Xclang`.)

Output (abbreviated):

```
TranslationUnitDecl 0x... <<invalid sloc>> <invalid sloc>
...
`-FunctionDecl 0x... </tmp/add.c:1:1, col:39> col:5 add 'int (int, int)'
  |-ParmVarDecl 0x... <col:9, col:13> col:13 used a 'int'
  |-ParmVarDecl 0x... <col:16, col:20> col:20 used b 'int'
  `-CompoundStmt 0x... <col:23, col:39>
    `-ReturnStmt 0x... <col:25, col:36>
      `-BinaryOperator 0x... <col:32, col:36> 'int' '+'
        |-ImplicitCastExpr 0x... <col:32> 'int' <LValueToRValue>
        | `-DeclRefExpr 0x... <col:32> 'int' lvalue ParmVar ... 'a' 'int'
        `-ImplicitCastExpr 0x... <col:36> 'int' <LValueToRValue>
          `-DeclRefExpr 0x... <col:36> 'int' lvalue ParmVar ... 'b' 'int'
```

The AST reflects the C abstract grammar faithfully: the `ImplicitCastExpr` nodes wrapping the `DeclRefExpr` are not written by the programmer but are inserted by Sema to represent the implicit lvalue-to-rvalue conversion that reading a variable entails. These implicit nodes are where Clang enforces C/C++ type rules before codegen ever sees the tree.

For C++, add `-ast-dump-filter <name>` to narrow the output to a specific declaration. Use `-ast-print` to reconstruct source from the AST — a round-trip that can expose simplifications.

The AST dump output changed subtly in LLVM 22: implicit `this` parameters now appear explicitly on method declarations in the dump (as `ParmVarDecl` with `implicit` annotation), and `DeducedTemplateSpecializationType` nodes carry the deduced type directly. These are cosmetic improvements for readability, not semantic changes.

ClangIR (`-fclangir`) inserts an optional CIR layer between the AST and LLVM IR. When enabled (the default Ubuntu 22 package has it disabled — rebuild with `-DCLANG_ENABLE_CIR=ON`), the codegen path goes AST → CIR MLIR dialect → either direct LLVM IR lowering or MLIR lowering through intermediate dialects. CIR preserves C/C++ structural idioms (classes with methods, reference types, `nullptr`, zero-initialisation semantics) that are typically lost in the AST → LLVM IR translation. [Chapter 52 covers ClangIR architecture; Chapter 53 covers CIR generation.]

### Stage 2: LLVM IR (Unoptimised)

```bash
$ /usr/lib/llvm-22/bin/clang -emit-llvm -S -o - /tmp/add.c
```

Output:

```llvm
; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @add(i32 noundef %0, i32 noundef %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, ptr %3, align 4
  store i32 %1, ptr %4, align 4
  %5 = load i32, ptr %3, align 4
  %6 = load i32, ptr %4, align 4
  %7 = add nsw i32 %5, %6
  ret i32 %7
}
```

At `-O0` (the default when no optimisation level is specified), the `optnone` attribute is attached and the IR is deliberately naive: parameters are stored to `alloca` slots and loaded back. This ensures that unoptimised builds are predictable and debuggable. The `mem2reg` pass, which is run early in any non-zero optimisation pipeline, promotes these alloca-load-store chains to SSA values.

The `noundef` attribute on parameters is an assertion that the caller will never pass poison or undef — it enables stronger inter-procedural optimisations. The `nsw` flag on `add nsw` asserts no-signed-wrap, enabling strength-reduction transformations that would be unsound for wrapping arithmetic.

### Stage 2b: Bitcode

```bash
$ /usr/lib/llvm-22/bin/clang -emit-llvm -c -o /tmp/add.bc /tmp/add.c
$ /usr/lib/llvm-22/bin/llvm-dis /tmp/add.bc -o -      # bitcode → text
$ /usr/lib/llvm-22/bin/llvm-as /tmp/add.ll  -o /tmp/add.bc  # text → bitcode
```

Bitcode is the binary encoding of LLVM IR. It is not an object file and not directly executable. The main use cases are LTO (the linker reads bitcode from `.o` files produced with `-flto`) and caching (bitcode is smaller and faster to parse than textual IR). The `llvm-bcanalyzer` tool can inspect bitcode structure at the record level.

### Stage 3: Optimised IR

```bash
$ /usr/lib/llvm-22/bin/clang -emit-llvm -S -O2 -o - /tmp/add.c
```

Output:

```llvm
; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local i32 @add(i32 noundef %0, i32 noundef %1) local_unnamed_addr #0 {
  %3 = add nsw i32 %1, %0
  ret i32 %3
}
```

After `mem2reg`, `instcombine`, and dead-code elimination, the stack traffic is gone. The attributes have changed dramatically: `mustprogress` (the function cannot loop forever), `memory(none)` (it neither reads nor writes memory), `nofree` (it does not call `free`), `nosync` (it is not affected by concurrency), `willreturn` (it terminates). These attributes are derived facts that the optimiser proved about the function — they enable further optimisations in callers.

`local_unnamed_addr` means the address of the function is not significant to the module (no global takes its address), allowing the linker to merge identical function bodies.

To trace the IR after each individual pass:

```bash
$ /usr/lib/llvm-22/bin/opt -passes='instcombine' -print-after-all -S /tmp/add_O1.ll
```

Output (excerpt):

```
; *** IR Dump After InstCombinePass on add ***
define dso_local i32 @add(i32 noundef %0, i32 noundef %1) local_unnamed_addr #0 {
  %3 = add nsw i32 %1, %0
  ret i32 %3
}
```

The pass name annotation in `*** IR Dump After ... ***` identifies the pass class. For complex pipelines with many passes, pipe through `grep "IR Dump"` to enumerate which passes fire and in which order.

To inspect the full O2 pipeline structure:

```bash
$ /usr/lib/llvm-22/bin/opt -passes='default<O2>' --print-pipeline-passes \
      -disable-output /tmp/add_O1.ll
```

This prints the complete, nested pass pipeline string. The O2 pipeline in LLVM 22 begins with `memprof-remove-attributes,annotation2metadata,forceattrs,inferattrs,coro-early` and proceeds through several nested CGSCC (call-graph SCC) iteration loops, including the main inliner (`inline`), `function-attrs`, `sroa`, `early-cse`, `instcombine`, loop transforms, vectorisers, and ultimately `verify`. The pipeline is 15+ levels of nesting when pretty-printed — studying its structure directly answers questions like "does loop vectorisation happen before or after the inliner?" (answer: vectorisation is inside the CGSCC devirtualise loop, so inlining happens first for each SCC, then vectorisation runs on the inlined body).

For targeted IR debugging, you can run a single pass in isolation:

```bash
# Run only mem2reg on -O0 IR, see the SSA promotion:
$ /usr/lib/llvm-22/bin/opt -passes='mem2reg' -S /tmp/add.ll

# Run mem2reg then instcombine:
$ /usr/lib/llvm-22/bin/opt -passes='mem2reg,instcombine' -S /tmp/add.ll
```

The new pass manager's `-passes` string is composable. `function(mem2reg)` runs `mem2reg` on each function separately (function pass). `module(globaldce)` runs `globaldce` on the whole module (module pass). `cgscc(inline)` runs `inline` on each strongly-connected component of the call graph. These wrappers enforce the pass scheduling contract and are required for correctness in multi-threaded compilation scenarios.

### Stage 4: MachineIR

MIR is the representation used by the instruction selector, register allocator, and all subsequent machine-level passes. To observe MIR just after instruction selection:

```bash
$ /usr/lib/llvm-22/bin/llc -O2 -stop-after=finalize-isel /tmp/add_O1.bc -o -
```

Output (body section):

```yaml
body:             |
  bb.0 (%ir-block.2):
    liveins: $edi, $esi

    %1:gr32 = COPY $esi
    %0:gr32 = COPY $edi
    %2:gr32 = nsw ADD32rr %1, %0, implicit-def dead $eflags
    $eax = COPY %2
    RET 0, $eax
```

MIR is a YAML-embedded text format. The `%N:gr32` notation indicates a virtual register constrained to the `gr32` register class (32-bit general-purpose registers on x86-64). Physical registers (`$edi`, `$eax`) appear at ABI boundaries — function entry and return. The `implicit-def dead $eflags` on `ADD32rr` records that the instruction defines the flags register but that value is known dead, so register allocation need not preserve it.

The `-stop-after=` mechanism accepts any pass name; `-stop-before=` stops before the named pass. The companion `-start-after=` and `-start-before=` flags allow re-entering the pipeline mid-stream, which is invaluable for narrowing bugs to specific passes. [Chapter 84 covers SelectionDAG; Chapter 85 covers GlobalISel and MIR.]

### Stage 5: Assembly and Object Code

```bash
$ /usr/lib/llvm-22/bin/llc /tmp/add.bc -o /tmp/add.s   # assembly
$ /usr/lib/llvm-22/bin/llvm-objdump -d /tmp/add.o       # disassemble object
```

Assembly output:

```asm
add:
        pushq   %rbp
        movq    %rsp, %rbp
        movl    %edi, -8(%rbp)
        movl    %esi, -4(%rbp)
        movl    -8(%rbp), %eax
        addl    -4(%rbp), %eax
        popq    %rbp
        retq
```

Object disassembly (note byte offsets):

```
0000000000000000 <add>:
       0: 55            pushq   %rbp
       1: 48 89 e5      movq    %rsp, %rbp
       4: 89 7d fc      movl    %edi, -0x4(%rbp)
       7: 89 75 f8      movl    %esi, -0x8(%rbp)
       a: 8b 45 fc      movl    -0x4(%rbp), %eax
       d: 03 45 f8      addl    -0x8(%rbp), %eax
      10: 5d            popq    %rbp
      11: c3            retq
```

The MC (Machine Code) layer is where assembly and binary emission are unified: the same `MCInst` stream can be printed as `.s` text, encoded as ELF `.text`, or encoded as Mach-O `__text`. The LLVM MC library ([`llvm/lib/MC/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/MC/)) is also used by `llvm-mc` for standalone assembly processing.

The progression from the `-O0` assembly (13 instructions, full frame setup) to the `-O2` assembly (2 instructions: `leal (%rdi,%rsi), %eax; retq` — a single `LEA` can replace `ADD` + `MOV` at -O2) illustrates the entire point of the pipeline: each stage transforms the representation in a way that exposes optimisation opportunities that would be invisible or incorrect to perform at the previous stage. The frame pointer in the `-O0` output is generated because the driver sets `-mframe-pointer=all` at `-O0` (for debuggability); at `-O2` the driver sets `-mframe-pointer=none` and the frame overhead disappears.

To verify the final object's symbol table and section layout:

```bash
# List all symbols in the object:
$ /usr/lib/llvm-22/bin/llvm-nm /tmp/add.o

# Dump section sizes and names:
$ /usr/lib/llvm-22/bin/llvm-size /tmp/add.o

# Full ELF header and section table:
$ /usr/lib/llvm-22/bin/llvm-readelf -a /tmp/add.o | head -40
```

---

## 3.5 The C/C++ Pipeline in Detail

The standard `clang foo.c -o foo` command executes the following stages internally:

```
foo.c ──(Preprocessor)──► foo.i ──(Lexer/Parser/Sema)──► AST
                                                             │
                                              (CodeGen) ─── ┘
                                                             │
                                                             ▼
                                                       LLVM IR (in-memory)
                                                             │
                                              (opt passes) ─┘
                                                             │
                                                             ▼
                                                      Optimised IR (in-memory)
                                                             │
                                              (llc / BackendJob) ─┘
                                                             │
                                                             ▼
                                                       foo.o (ELF object)
                                                             │
                                              (ld / lld) ─── ┘
                                                             │
                                                             ▼
                                                         foo (ELF binary)
```

In a single `clang` invocation, these steps all happen in the same process (for `-O0`) or as a small number of sub-processes. The driver stages are:

1. **Preprocess** (`-E`): The preprocessor handles `#include`, `#define`, `#if`, and `#pragma`. It communicates with the lexer through the `Token` stream.
2. **Compile** (`-S` or `-c`): cc1 runs the full frontend (lexer → parser → Sema → CodeGen) and the middle-end (opt pipeline). With `-emit-llvm -S`, it stops before the backend and emits textual IR.
3. **Assemble** (implicit, unless `-S`): The MC layer converts the instruction stream to a relocatable object file. With `-c`, Clang stops here.
4. **Link** (the final step unless `-c`): The linker resolves symbol references, applies relocations, and writes the final ELF/Mach-O/PE binary.

The object file produced at step 3 contains `.text`, `.rodata`, `.data`, DWARF debug sections (if `-g`), and an address-significance table (`.llvm_addrsig`) if `-faddrsig` is active (it is by default). The address-significance table allows lld's identical-code folding (ICF) to be safe: functions not in the table may be freely merged.

For C++, an additional step — **template instantiation** — occurs inside Sema before codegen. The IR that CodeGen sees is fully monomorphised: all templates have been instantiated, all `constexpr` expressions evaluated, all overloads resolved. LLVM IR has no notion of templates or overloads. This is fundamentally different from languages like Swift or Rust, which carry generic/trait information deeper into the pipeline before monomorphisation.

### Deferred Emission

Clang uses **deferred emission** for functions that might not be needed. When the parser encounters a function definition inside a class body (an implicit `inline` function), Clang does not immediately generate IR. Instead, it stores the AST node and emits it lazily only when a caller is encountered that actually needs the definition. This lazy emission is managed by `CodeGenModule::EmitDeferredDecls()` in [`clang/lib/CodeGen/CodeGenModule.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenModule.cpp). It is a significant compile-time optimisation for large C++ headers where most inline functions in a class are never called from a given translation unit.

### The `-save-temps` Flag

To inspect all intermediate files without writing custom pipelines:

```bash
$ /usr/lib/llvm-22/bin/clang -save-temps -O2 -c /tmp/add.c
ls /tmp/add.*
# add.c  add.i  add.bc  add.s  add.o
```

`-save-temps` preserves every intermediate file: `.i` (preprocessed source), `.bc` (unoptimised bitcode), `.s` (final assembly before object encoding). The `.bc` file is the IR *after* the frontend but *before* the opt pipeline — note that the term "unoptimised" is a slight misnomer because `-O2` is also passed to cc1, which drives the opt pipeline. The `.s` file is the final assembly fed to the MC layer for object emission. This is often the fastest way to see "what IR was generated" without constructing a multi-step pipeline manually.

---

## 3.6 LTO and ThinLTO Pipelines

Link-time optimisation allows the compiler to see across translation-unit boundaries. The two modes differ in how much work is done at link time.

### Monolithic LTO (`-flto`)

```
foo.c ──(cc1 -flto)──► foo.o  [contains LLVM bitcode, not native code]
bar.c ──(cc1 -flto)──► bar.o  [contains LLVM bitcode]
                                │
                    (linker plugin or lld) reads bitcode
                                │
                                ▼
                     merged LLVM IR module
                                │
                     full optimization pipeline
                                │
                                ▼
                         native code + linked binary
```

With `-flto`, cc1 uses `-emit-llvm-bc` internally. The `.o` file produced is not a native object — it is an ELF container wrapping LLVM bitcode. The system linker's plugin interface (LLVMgold.so for GNU ld, or lld's native support) extracts the bitcode, links all modules together, runs the full optimiser on the merged IR, then generates code. The `clang -### -flto` output reveals the `-plugin /usr/lib/llvm-22/lib/LLVMgold.so` argument passed to the linker.

The advantage: maximum interprocedural optimisation visibility — inlining across `.c` file boundaries, dead function elimination, whole-program devirtualisation. The disadvantage: link time proportional to the total IR size of all translation units, with serial optimisation being the bottleneck.

### ThinLTO (`-flto=thin`)

ThinLTO preserves cross-unit visibility at a fraction of the link-time cost:

```
foo.c ──(cc1 -flto=thin)──► foo.o  [bitcode + per-module summary]
bar.c ──(cc1 -flto=thin)──► bar.o  [bitcode + per-module summary]
                                    │
                      linker reads summaries (fast)
                                    │
                        constructs call-graph, import lists
                                    │
                      ┌─────────────┴─────────────┐
                      │  Import foo→bar symbols   │  (parallel)
                      │  Import bar→foo symbols   │
                      └─────────────┬─────────────┘
                                    │
                      per-module optimisation + codegen
                                    │  (parallel, one module per core)
                                    ▼
                           native objects + final link
```

The key innovation is the **module summary index** (stored in the bitcode's summary section). It encodes call graph edges, inlinability hints, and global variable access patterns in a compact form. The linker reads only the summaries to determine which definitions need to be imported into each module, then imports just those functions. Each module is optimised and compiled independently and in parallel.

ThinLTO typically achieves 70-90% of monolithic LTO's speedup at a fraction of the link-time cost, with near-perfect parallelism. The `-flto=thin` cc1 invocation includes `-flto-unit` (produce the summary) and `-vectorize-loops -vectorize-slp` (ThinLTO benefits from vectorisation feedback in the summary).

The summary-based import decision uses the `FunctionSummary::ImportThreshold` heuristic: functions below a code-size threshold are candidates for importing. The threshold is exposed via `-import-instr-limit`.

### Monolithic vs ThinLTO: Which to Use

The choice between LTO modes depends on the build's bottleneck. Monolithic LTO is appropriate for release builds of modest-size programs where link time is acceptable and maximum performance is required. ThinLTO is appropriate for large codebases (Chromium, LLVM itself, Firefox) where monolithic LTO's serial link step would be prohibitive. For developer iteration builds, neither LTO mode is typically appropriate — LTO is a release-build optimisation.

A practical note: the `clang -### -flto=thin` output on this system shows `-plugin /usr/lib/llvm-22/lib/LLVMgold.so` being passed to `ld` (the GNU linker). lld has native ThinLTO support that is faster and more robust — use `-fuse-ld=lld -flto=thin` in production builds where lld is available. The performance difference in link time between gold+LLVMgold.so and lld is typically 2–4x in lld's favour for large codebases.

### Distributed ThinLTO

A third LTO variant not exposed via `-flto=thin` directly is **distributed ThinLTO** (dthinlto), designed for build systems that distribute work across many machines. In dthinlto, the linker's role is reduced to producing a distribution plan (import lists per module); individual modules are then optimised and compiled on worker machines. The final link step assembles the pre-compiled native objects. This is used internally at some large tech companies for linking programs with millions of source files. The `llvm-lto2` tool exposes dthinlto operations at the command line for testing.

---

## 3.7 Offload Compilation: CUDA, HIP, OpenMP

GPU compilation is architecturally different from CPU compilation because a single source file produces code for two distinct targets: the **host** (x86-64, AArch64, etc.) and one or more **devices** (NVIDIA PTX, AMD GCN, etc.).

### The Single-Source Split

```
  kernel.cu (CUDA) / kernel.hip (HIP)
         │
   clang driver (offload model)
         │
    ┌────┴────────────────────────────────────┐
    │  Host cc1 pass                          │  Device cc1 pass
    │  triple: x86_64-pc-linux-gnu            │  triple: nvptx64-nvidia-cuda
    │  Kernel calls → runtime stubs           │  Kernel body → PTX / GCN ISA
    ▼                                         ▼
  host LLVM IR                          device LLVM IR
    │                                         │
    │  host opt pipeline                      │  device opt pipeline
    ▼                                         ▼
  host .o                               device PTX/cubin/hsaco
    │                                         │
    └──────────────┬──────────────────────────┘
                   │  clang-offload-bundler / fat binary embedding
                   ▼
         host .o with device code embedded
                   │
             host linker
                   │
                   ▼
           host binary with embedded fatbinary
```

**CUDA path.** Clang compiles `.cu` files twice. In the device pass, CUDA built-ins (`__syncthreads`, `threadIdx`, etc.) are resolved against CUDA's `libdevice.bc` (a bitcode library of device math functions). The device IR is lowered to PTX (via the NVPTX backend) or to cubin (via ptxas invoked separately). The fatbinary contains PTX, SASS (native GPU instructions), or both.

**HIP path.** Structurally identical to CUDA but targets AMDGPU GCN via the AMDGPU backend. The device objects are in the ROCm code object format (`.hsaco`), and the device math library is `rocm-device-libs`. The offload bundler (`clang-offload-bundler`) packages host and device code into a single object file using the `__CLANG_OFFLOAD_BUNDLE__` section scheme.

**OpenMP target offload.** The OpenMP model introduces a third compilation step: the target region (the block after `#pragma omp target`) is extracted, compiled for the device, and packaged similarly to CUDA/HIP. The device image is linked into the host binary as data, and the OpenMP runtime (`libomptarget`) locates and loads it at run time.

Inspecting offload intermediate representations requires additional flags. For CUDA/PTX:

```bash
$ clang --cuda-device-only -emit-llvm -S -o device.ll kernel.cu
```

For HIP/GCN:

```bash
$ clang --hip-device-only -emit-llvm -S -o device.ll kernel.hip
```

The `clang-offload-bundler --list` command lists the target triples bundled into an offload object. [Chapter 48 covers Clang as a CUDA compiler; Chapter 49 covers HIP; Chapter 50 covers SYCL and OpenMP offload.]

### The New Offload Driver Model

LLVM 14 introduced the new offload driver model (`-offload-new-driver`, now enabled by default in LLVM 22 for most targets). In the new model, the driver creates separate compilation jobs for each offload target without the `clang-offload-bundler` tool; instead, a new `clang-linker-wrapper` tool wraps the host linker and handles device linking, device-code optimisation, and embedding. The `llvm-offload-binary` tool replaces `clang-offload-bundler` for the new format. The old bundler format is still supported for compatibility with ROCm 5.x toolchains.

Key differences in the new model:

| Aspect | Old model (bundler) | New model (linker-wrapper) |
|---|---|---|
| Device link step | Separate `clang-offload-bundler` | `clang-linker-wrapper` wraps host linker |
| Device LTO | Manual `-lto` flags | Automatic via wrapper |
| Object format | `__CLANG_OFFLOAD_BUNDLE__` ELF section | Separate `.o` per device, embedded at link |
| Compatibility | ROCm ≤ 5.x, CUDA traditional | ROCm ≥ 5.5, CUDA new-offload |

The NVPTX target (NVIDIA) remains the canonical example of the device ISA path. LLVM's NVPTX backend emits PTX (Parallel Thread eXecution) — NVIDIA's virtual ISA — which is then compiled to SASS (the actual GPU instruction set) by `ptxas` at load time (or offline via `cuobjdump`). This two-stage device compilation is analogous to the two-stage host compilation: PTX is to SASS what LLVM IR is to x86 machine code.

---

## 3.8 The MLIR and Flang Pipelines

### The MLIR Pipeline

MLIR does not have a single fixed pipeline. Instead, it provides a **dialect infrastructure** and a pass manager, and each project that uses MLIR defines its own progression of dialects from high abstraction to low. The canonical toy-to-hardware path for a matrix computation might be:

```
  High-level dialect (Linalg / Tensor / Affine)
         │
   mlir-opt --linalg-to-loops           (tiling, fusion)
         │
   mlir-opt --lower-affine              (Affine → SCF → Standard)
         │
   mlir-opt --convert-scf-to-cf        (structured control flow → CFG)
         │
   mlir-opt --convert-func-to-llvm --convert-arith-to-llvm
         ▼
   LLVM dialect (llvm.func, llvm.add, llvm.alloca, ...)
         │
   mlir-translate --mlir-to-llvmir
         ▼
   LLVM IR (.ll)
         │
   llc / opt (standard LLVM backend)
         ▼
   object code
```

Each `mlir-opt` invocation is either a standalone pipeline step or a multi-pass pipeline specified with `--pass-pipeline`. The pass pipeline syntax nests: `builtin.module(func.func(canonicalize,cse),inline)` runs `canonicalize` and `cse` on each function, then inlines.

Verifying the working MLIR-to-LLVM-IR translation for our `add` function:

```bash
$ cat add.mlir
func.func @add(%a: i32, %b: i32) -> i32 {
  %c = arith.addi %a, %b : i32
  return %c : i32
}

$ /usr/lib/llvm-22/bin/mlir-opt add.mlir \
    --convert-func-to-llvm \
    --convert-arith-to-llvm \
    --reconcile-unrealized-casts | \
  /usr/lib/llvm-22/bin/mlir-translate --mlir-to-llvmir
```

Output:

```llvm
; ModuleID = 'LLVMDialectModule'
source_filename = "LLVMDialectModule"

define i32 @add(i32 %0, i32 %1) {
  %3 = add i32 %0, %1
  ret i32 %3
}
```

Note that the MLIR-derived IR lacks the `nsw` flag (the arith dialect's `addi` does not assert signed-no-overflow by default) and the `noundef` attribute (the func dialect does not have C-language semantics for argument poison). This is not a deficiency — these attributes encode source-language guarantees that only a C frontend can legitimately assert.

The `--reconcile-unrealized-casts` pass is required to eliminate unrealised cast operations that remain after partial lowering. It is a standard cleanup pass in the lowering chain. The MLIR pass manager (`mlir-opt -print-ir-after-all`) is the direct analog of `opt -print-after-all`.

### MLIR Pass Pipelines: Structure and Debugging

MLIR's pass infrastructure is described in [`mlir/include/mlir/Pass/PassManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Pass/PassManager.h). The pass manager in MLIR enforces an **op-specificity contract**: a pass registered to operate on `func.func` operations only sees `func.func` ops, not `builtin.module` or `llvm.func`. This prevents passes from accidentally walking outside their intended scope and makes parallelism over function-level passes safe by construction.

The `--pass-pipeline` syntax mirrors this structure:

```bash
# Canonicalize at module level, then CSE inside each function:
$ /usr/lib/llvm-22/bin/mlir-opt add.mlir \
    --pass-pipeline="builtin.module(canonicalize,func.func(cse))"

# Show IR after every pass (MLIR analog of opt -print-after-all):
$ /usr/lib/llvm-22/bin/mlir-opt add.mlir \
    --convert-arith-to-llvm \
    --print-ir-after-all 2>&1 | head -30
```

MLIR's verifier runs automatically after every pass (in debug builds) and can be manually invoked with `--verify-each`. This catch-as-you-go verification is a major reliability advantage over LLVM IR, where `opt -verify` must be explicitly inserted in the pipeline. MLIR's type system and op verification contracts make many classes of IR corruption detectable immediately at the pass that introduced them.

### The Flang Pipeline

Flang (the LLVM Fortran compiler, `flang-new` in the source tree) introduces two additional intermediate representations before LLVM IR:

```
  .f90 source
       │
  flang-new -fc1 (parsing + semantic analysis)
       │
  Fortran parse tree → Fortran semantics (symbol table)
       │
  flang-new -fc1 -emit-hlfir   (High-Level FIR)
       ▼
  HLFIR — High Level Fortran IR
       │   (array syntax, Fortran intrinsics, intent-in/out semantics)
  flang-new -fc1 -emit-fir     (lower HLFIR → FIR)
       ▼
  FIR — Fortran IR (MLIR dialect: fir.*, hlfir.*)
       │   (explicit memory, loops, boxed arrays, character strings)
  tco (the Flang MLIR-to-LLVM lowerer)
       │   (FIR → LLVM dialect → mlir-translate)
       ▼
  LLVM IR
       │
  (standard llc/opt pipeline)
       ▼
  object code
```

**HLFIR** (High-Level FIR) preserves Fortran-level semantics: array sections (`A(1:N)`), elemental intrinsics (`SUM`, `MATMUL`), and `INTENT(IN/OUT)` annotations. It is the level at which Fortran-specific optimisations — array copy elimination, loop fusion for array expressions — are applied.

**FIR** (Fortran IR) is a lower-level MLIR dialect where array references are explicit memory operations and Fortran's boxed-array type (`!fir.box`) represents the array descriptor (shape, stride, base pointer). FIR closely mirrors the execution model of the Fortran standard but is still above the LLVM IR level.

The `tco` tool (Tilikum Crossing Optimizer — the historical name for the Flang MLIR pipeline driver) runs the FIR→LLVM lowering sequence. In the LLVM 22 source tree, `tco` invokes `fir::createMLIRToLLVMPassPipeline()` which chains FIR canonicalisation, memory reuse, loop restructuring, and finally `mlir-to-llvmir`. [Chapter 87 — Flang Internals covers the FIR and HLFIR dialects in depth.]

Flang is not installed in the standard `llvm-22` Ubuntu package; to experiment with it, build from source with `-DLLVM_ENABLE_PROJECTS="clang;flang"` and the Flang runtime. The canonical `flang-new -fc1 -emit-fir -o add.fir add.f90` command is the analog of `clang -emit-llvm -S`.

A simple Fortran example:

```fortran
! add.f90
integer function add(a, b)
  integer, intent(in) :: a, b
  add = a + b
end function add
```

Would produce FIR roughly as:

```mlir
func.func @add_(%arg0: !fir.ref<i32>, %arg1: !fir.ref<i32>) -> i32 {
  %0 = fir.load %arg0 : !fir.ref<i32>
  %1 = fir.load %arg1 : !fir.ref<i32>
  %2 = arith.addi %0, %1 : i32
  return %2 : i32
}
```

The `!fir.ref<i32>` type (a typed reference — Fortran passes by reference by default) is the key difference from the MLIR version: Fortran's calling convention is pass-by-reference, so the FIR representation preserves the references before lowering them to LLVM's pointer-based loads. The HLFIR layer above this would represent the Fortran INTENT semantics more explicitly, allowing analyses to reason about aliasing and mutation without yet committing to pointer operations. [Chapter 87 covers the Flang pipeline in depth, including HLFIR transformations and the FIR→LLVM lowering sequence.]

---

## 3.9 JIT Compilation with ORC

The ORC (On-Request Compilation) JIT framework is the LLVM JIT infrastructure introduced in LLVM 5 and made the default in LLVM 12. It supersedes the older MCJIT framework. ORC's design principle is that all JIT operations are **asynchronous symbol lookups** across a hierarchy of `JITDylib` (dynamic library) objects.

### ORC Architecture

```
  LLVM IR Module (in-memory)
         │
  LLJIT / ORCJit::addModule()
         │
  IRCompileLayer       ← compiles IR to native machine code (ThreadSafeModule)
         │
  RTDyldObjectLinkingLayer  ← loads and links the object in-memory
         │                    resolves external symbols via JITDylib lookup
         │
  JITDylib ("main")   ← symbol table: maps function names → addresses
         │
  ExecutionSession     ← owns all JITDylibs, drives async lookup
         │
  MaterializationUnit  ← on-demand unit: materialises definitions when looked up
         │
  TargetProcessControl ← controls the execution context
         │
         ▼
  In-process execution: call function pointer
```

ORC supports three execution models:

1. **In-process JIT**: IR is compiled and executed in the same process. The classic use case. The `lli` tool runs this way.
2. **Out-of-process JIT**: The JIT and executor run in different processes connected by a socket or pipe. Useful for JIT-compiling code that must run as a different user, in a sandbox, or on a remote device.
3. **Remote execution**: The executor runs on different hardware (e.g., compiling for a GPU or embedded device, then invoking the result via an RPC mechanism).

The simplest ORC invocation is `lli`, which JIT-compiles and runs a `.ll` file:

```bash
$ /usr/lib/llvm-22/bin/lli /tmp/add_main.ll
add(3,4)=7
```

`lli` creates an `LLJIT` instance, adds the module to the `main` JITDylib, looks up the `@main` symbol, and calls the resulting function pointer. The JIT compiles the module on the first symbol lookup — this is the "on-request" in ORC.

### Lazy Compilation

ORC's most powerful capability is **lazy compilation**: function bodies are compiled only when first called.

```
  Module added to LLJIT (all functions registered as stubs)
         │
  Stub function pointer returned immediately (no compilation yet)
         │
  First call to function pointer hits the stub
         │
  Stub triggers lazy compilation of that function body only
         │
  Stub updated to point to compiled code
         │
  Subsequent calls go directly to compiled code
```

This is implemented via `LazyReexportsMaterializationUnit`. Each stub is a small trampoline that calls back into ORC's `MaterializationDispatch` mechanism. The first call triggers IR-to-native compilation for just that function; the trampoline is atomically updated to point to the result.

Lazy compilation is fundamental to language runtime JITs (LLVM's ORC is used in Swift, Julia, and LLDB's expression evaluator) where compiling every function eagerly would impose unacceptable startup latency.

### The llvm-jitlink Tool

`llvm-jitlink` is a lower-level JIT tool that bypasses the IR layer and loads native object files (`.o`, `.so`) directly into ORC's linking layer. It is primarily a test harness for ORC's `JITLink` infrastructure (the new, platform-portable linking layer introduced in LLVM 11 to replace `RuntimeDyld`). `llvm-jitlink` also serves as a useful tool for verifying that an object file links and executes correctly without constructing a full process environment. [Chapter 74 covers ORC JIT internals in full.]

### Language Runtime JIT Use Cases

Three major language runtimes built on ORC JIT illustrate its breadth:

**Swift.** The Swift compiler uses ORC to power the REPL and the `#preview` macro in Xcode. When a Swift function is called for the first time in a REPL session, ORC compiles it, resolves against the session's symbol table (which includes all previously compiled declarations), and executes the result. The session's symbol table grows with each REPL evaluation, and all previously compiled code remains resident.

**Julia.** Julia's entire execution model is ORC-based: every method call dispatch path through Julia's multiple-dispatch type system can trigger on-demand JIT compilation of a specialised method body. Julia has contributed significantly to ORC's scalability — the ability to manage tens of thousands of small compilation units with fast symbol lookup.

**LLDB expression evaluator.** When you type `p expr` in LLDB, the debugger compiles the expression using Clang, then JIT-compiles the IR into the debugged process's address space via ORC's out-of-process JIT mode. The expression can access all symbols in the inferior process and return results to the debugger. This is a non-trivial use of ORC: the execution context is a remote process with a different address space, symbol tables must be reconstructed from DWARF, and the compilation must complete in well under a second for interactive use.

---

## 3.10 Intermediate Representations: A Comparative Summary

Each IR layer serves a distinct purpose and has properties that make it the right representation for its stage.

| Layer | Representation | Key properties | Primary tools |
|---|---|---|---|
| Source | C/C++/Fortran text | Human-readable, language semantics | Text editor, clang -E |
| AST | In-memory tree | Faithful to source; carries types, source locations | -Xclang -ast-dump |
| CIR (ClangIR) | MLIR dialect | Optional; higher-level than LLVM IR; C/C++ idioms visible | -fclangir -emit-cir |
| LLVM IR | SSA, infinite registers | Target-independent; rich type system; opaque pointers | clang -emit-llvm, opt, llvm-dis |
| SelectionDAG | DAG per BB | Type-legalised; target-specific ops; combines and patterns | -debug-only=isel, graphviz views |
| Generic MIR | SSA, typed vregs | GlobalISel path; target-generic machine opcodes | -stop-after=legalizer |
| MIR | Physical/virtual regs | Post-selection; explicit register classes; stack frame | -stop-after=finalize-isel |
| MC | MCInst stream | Target-specific encoding; no LLVM types | llvm-mc, llc -filetype=asm |
| Object | ELF/Mach-O/COFF | Relocatable binary + debug sections | llvm-objdump, llvm-readobj |
| Binary | Linked executable | Absolute addresses; PLT/GOT resolved | llvm-objdump, llvm-symbolizer |

The richness of LLVM's observable intermediate layers is a design virtue: every stage can be serialised to a file, inspected with a standard tool, and re-injected mid-pipeline. This property makes LLVM exceptionally amenable to compiler research, automated testing with `FileCheck`, and incremental debugging of correctness bugs.

---

## 3.11 Chapter Summary

- The LLVM compilation pipeline transforms source through eight observable layers: preprocessed source, AST, LLVM IR, (optionally) SelectionDAG or generic MIR, MachineIR, MC, object file, and linked binary.
- The `clang` binary is a **driver**; `clang -cc1` is the actual compiler. Use `clang -###` to reveal the exact cc1 invocation and understand how driver flags map to compiler flags.
- `opt` owns the middle-end IR-to-IR transformation. In LLVM 22 it uses only the new pass manager; pipelines are specified with `-passes='...'`. Use `-print-after-all` to trace IR evolution pass-by-pass.
- `llc` owns the backend: instruction selection, register allocation, and MC emission. Use `-stop-after=finalize-isel` to dump MIR immediately after instruction selection.
- `llvm-as`/`llvm-dis` provide lossless round-trip between textual `.ll` and binary `.bc`. Bitcode is the currency of LTO.
- `llvm-link` merges IR modules; `llvm-ar` manages bitcode archives. Neither tool does codegen.
- LTO (`-flto`) maximises interprocedural optimisation by deferring codegen to link time. ThinLTO (`-flto=thin`) achieves most of the benefit with parallel, scalable link-time compilation via module summaries and selective importing.
- GPU offload (CUDA, HIP, OpenMP target) splits compilation into host and device passes, then packages both into a single object via the offload bundler. The device pass targets NVPTX or AMDGCN and links against device math libraries.
- MLIR pipelines are user-defined sequences of dialect lowerings, each observable with `mlir-opt`. The terminal step is `mlir-translate --mlir-to-llvmir`, which produces standard LLVM IR for the backend.
- Flang introduces FIR and HLFIR above LLVM IR, preserving Fortran semantics (array sections, intrinsics, intent annotations) through the optimisation stages before lowering to the LLVM backend.
- ORC JIT compiles IR or objects on-demand in-process, out-of-process, or remotely. Lazy compilation defers function-body compilation until first call, enabling interactive REPLs and language runtimes with fast startup.
- Every IR layer is observable with a one-liner: `-E` for preprocessed source, `-Xclang -ast-dump` for AST, `-emit-llvm -S` for IR, `-stop-after=finalize-isel` for MIR, `llvm-objdump -d` for final machine code.


---

@copyright jreuben11
