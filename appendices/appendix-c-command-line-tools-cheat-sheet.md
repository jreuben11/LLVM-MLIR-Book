# Appendix C — Command-Line Tools Cheat Sheet

*Quick Reference | LLVM 22.1.x*

All tools are located at `/usr/lib/llvm-22/bin/` on Debian/Ubuntu package installs, or `$LLVM_BUILD/bin/` in a custom build. Prepend the path or add to `PATH`. For full documentation run `<tool> --help`.

---

## C.1 Compiler Tools

### `clang` / `clang++`

| Flag | Purpose |
|---|---|
| `-c` | Compile to object file only |
| `-S` | Compile to assembly (`.s`) |
| `-emit-llvm` | Emit LLVM IR (`.ll` with `-S`, `.bc` with `-c`) |
| `-emit-ast` | Emit Clang AST (`.ast`) |
| `-O0/1/2/3/Os/Oz/Og` | Optimization levels |
| `-g` / `-gline-tables-only` | Debug info (full / line tables only) |
| `-fPIC` / `-fPIE` | Position-independent code/executable |
| `-fno-exceptions` / `-fno-rtti` | Disable C++ exceptions/RTTI |
| `-fsanitize=address,undefined,...` | Enable sanitizers |
| `-fprofile-generate` / `-fprofile-use=file` | Instrumentation PGO |
| `-fcs-profile-generate` | Context-sensitive PGO |
| `-flto` / `-flto=thin` | Full LTO / ThinLTO |
| `-march=native` / `-march=x86-64-v3` | Target architecture |
| `-mllvm <flag>` | Pass flag directly to LLVM backend |
| `-Xclang <flag>` | Pass flag to Clang cc1 |
| `-v` | Verbose: show all subcommands |
| `-###` | Dry-run: show commands without executing |
| `-target <triple>` | Explicit target triple |
| `-isysroot <path>` | Sysroot for cross compilation |
| `-stdlib=libc++` | Use libc++ instead of libstdc++ |
| `-std=c++23` / `-std=c17` | Language standard |
| `-MF <file> -MMD` | Dependency file generation |
| `-fplugin=<so>` | Load Clang plugin |
| `-fpass-plugin=<so>` | Load LLVM pass plugin (new PM) |
| `-fvisibility=hidden` | Default symbol visibility |
| `-fstack-protector-strong` | Stack canary |
| `-fcoverage-mapping` | Source-based coverage instrumentation |
| `-frewrite-map-file=<f>` | Apply symbol rewrite rules |
| `-ftime-trace` | Emit Chrome tracing JSON for compilation phases |

**Common patterns:**
```bash
# Compile and emit readable IR
clang -O2 -S -emit-llvm -o out.ll input.c

# Cross-compile for AArch64
clang -target aarch64-linux-gnu -c -o out.o input.c

# Full ThinLTO build
clang -O3 -flto=thin -fuse-ld=lld -o prog main.c lib.c

# Debug with ASAN + UBSAN
clang -g -O1 -fsanitize=address,undefined -o prog main.c
```

### `clang-cl`

Drop-in MSVC-compatible driver. Accepts `/W3`, `/MT`, `/LD`, `/Fe:`, `/Fo:` flags. Use `--target=x86_64-windows-msvc` for cross.

---

## C.2 IR Tools

### `opt`

LLVM pass runner. Runs passes on `.ll` or `.bc` files.

| Flag | Purpose |
|---|---|
| `-S` | Output human-readable `.ll` |
| `-o <file>` | Output file |
| `-p <passes>` | New PM pipeline string (e.g. `"default<O2>"`, `"mem2reg,gvn"`) |
| `--passes=<pipeline>` | Alias for `-p` |
| `--print-before=<pass>` | Print IR before named pass |
| `--print-after=<pass>` | Print IR after named pass |
| `--print-pass-pipeline` | Print resolved pass pipeline |
| `--debug-pass-manager` | Verbose pass manager tracing |
| `--verify-each` | Verify IR after every pass |
| `--stats` | Print pass statistics |
| `--time-passes` | Time each pass |
| `--disable-output` | Run passes but discard output |
| `--load-pass-plugin=<so>` | Load out-of-tree pass plugin |
| `--passes=print<domtree>` | Print analysis result |
| `-march=<arch>` | Target for target-specific analyses |

```bash
# Run O2 pipeline
opt -S -p "default<O2>" input.ll -o out.ll

# Run mem2reg + GVN
opt -S --passes="mem2reg,gvn" input.ll -o out.ll

# Print IR before and after inliner
opt -S --print-before=inline --print-after=inline -p inline input.ll -o /dev/null
```

### `llc`

LLVM static compiler (IR → assembly/object).

| Flag | Purpose |
|---|---|
| `-march=<arch>` | Target architecture |
| `-mcpu=<cpu>` | Target CPU |
| `-mattr=<+feat,...>` | CPU feature flags |
| `-O0/1/2/3` | Optimization level |
| `-filetype=asm/obj/null` | Output type |
| `-o <file>` | Output file |
| `--print-before=<pass>` | Print MIR before pass |
| `--regalloc=<basic\|greedy\|pbqp>` | Register allocator |
| `--code-model=<small\|medium\|large>` | Code model |
| `--relocation-model=<static\|pic\|ropi>` | Relocation model |
| `-debug-only=<component>` | Debug output for component |
| `--mtriple=<triple>` | Full target triple |

```bash
llc -march=x86-64 -mcpu=znver4 -O2 -filetype=obj -o out.o input.ll
llc -march=aarch64 --mattr=+sve -O2 -filetype=asm input.ll
```

### `llvm-as` / `llvm-dis`

| Tool | Usage |
|---|---|
| `llvm-as` | `llvm-as input.ll -o output.bc` — assemble text IR to bitcode |
| `llvm-dis` | `llvm-dis input.bc -o output.ll` — disassemble bitcode to text IR |

### `llvm-link`

Link multiple IR modules: `llvm-link a.ll b.ll -S -o combined.ll`

Key flags: `--only-needed` (link only needed symbols), `--internalize`, `--override=<file>` (override with definitions from file).

### `llvm-extract`

Extract functions/globals from a module: `llvm-extract --func=foo input.ll -S -o foo.ll`

Key flags: `--func=<name>`, `--glob=<name>`, `--recursive`, `--keep-const-init`, `--delete` (delete instead of extract).

---

## C.3 Object and Binary Tools

### `llvm-ar`

LLVM archiver (compatible with GNU `ar`).

```bash
llvm-ar rcs libfoo.a a.o b.o c.o   # create static library
llvm-ar t libfoo.a                  # list contents
llvm-ar x libfoo.a                  # extract all
llvm-ar d libfoo.a old.o            # delete member
```

### `llvm-nm`

List symbols: `llvm-nm [--demangle] [--defined-only] [--extern-only] file`

| Output column | Meaning |
|---|---|
| Address | Symbol value |
| Type | `T`=text, `D`=data, `B`=bss, `U`=undefined, `W`=weak, `t/d/b`=local |
| Name | Symbol name |

Flags: `--format=posix/bsd/sysv`, `--radix=d/o/x`, `--print-size`, `--print-armap` (archive index), `--dynamic` (dynamic symbols).

### `llvm-objdump`

```bash
llvm-objdump -d prog                 # disassemble
llvm-objdump -d --symbolize prog     # with symbols
llvm-objdump -d -M intel prog        # Intel syntax
llvm-objdump -S prog                 # interleave source (requires debug info)
llvm-objdump -x prog                 # print all headers
llvm-objdump -r prog                 # show relocations
llvm-objdump --dwarf=info prog       # DWARF info
```

### `llvm-objcopy`

```bash
llvm-objcopy --strip-debug prog prog.stripped
llvm-objcopy --add-section .notes=notes.bin prog
llvm-objcopy --rename-section .data=.mydata prog
llvm-objcopy -O binary prog prog.bin    # extract raw binary
llvm-objcopy --set-section-flags .bss=alloc,load prog
```

### `llvm-strip`

`llvm-strip [-s] [-x] [-S] [-g] prog` — remove symbol table (`-s`), local symbols (`-x`), debug sections (`-S`/`-g`).

### `llvm-size`

`llvm-size [-A] [-B] file` — print section sizes. `-A`=System V format, `-B`=Berkeley format.

### `llvm-readelf` / `llvm-readobj`

```bash
llvm-readelf -h prog             # ELF header
llvm-readelf -S prog             # section headers
llvm-readelf -s prog             # symbol table
llvm-readelf -r prog             # relocations
llvm-readelf -d prog             # dynamic section
llvm-readelf -n prog             # notes
llvm-readelf --debug-dump=info prog  # DWARF debug info

llvm-readobj --all prog          # all information (machine-readable)
llvm-readobj --file-headers --sections prog
```

---

## C.4 Debug Info Tools

### `llvm-dwarfdump`

```bash
llvm-dwarfdump prog                      # all DWARF sections
llvm-dwarfdump --debug-info prog         # .debug_info only
llvm-dwarfdump --debug-line prog         # .debug_line only
llvm-dwarfdump --debug-frame prog        # .debug_frame
llvm-dwarfdump --debug-aranges prog
llvm-dwarfdump --debug-names prog        # DWARF 5 name index
llvm-dwarfdump --verify prog             # verify DWARF consistency
llvm-dwarfdump --lookup=0x401000 prog    # look up address
llvm-dwarfdump --name=foo prog           # find by name
llvm-dwarfdump --statistics prog         # debug quality stats
```

### `llvm-symbolizer`

Translate addresses to source locations:
```bash
llvm-symbolizer -e prog 0x401000
echo "prog 0x401000" | llvm-symbolizer
llvm-symbolizer --functions --demangle --inlining -e prog < addresses.txt
```

Flags: `--output-style=GNU/LLVM/JSON`, `--print-source-context-lines=N`, `--relativenames`.

### `llvm-dwp`

Merge `.dwo` files into a `.dwp` package: `llvm-dwp -e prog -o prog.dwp`

---

## C.5 Profile Tools

### `llvm-profdata`

```bash
# Merge raw profile files
llvm-profdata merge -o merged.profdata *.profraw

# Show profile summary
llvm-profdata show merged.profdata

# Show specific function counters
llvm-profdata show --function=foo merged.profdata

# Merge with weighted overlap
llvm-profdata merge -weighted-input=2,run1.profdata -weighted-input=1,run2.profdata -o out.profdata

# Convert instrumentation profile to indexed
llvm-profdata merge -instr -output=out.profdata in.profdata
```

Subcommands: `merge`, `show`, `overlap`, `order`.

### `llvm-cov`

```bash
# Show coverage report
llvm-cov report -instr-profile=prog.profdata prog

# Generate HTML
llvm-cov show -instr-profile=prog.profdata -format=html -output-dir=./coverage prog

# Export LCOV data
llvm-cov export -instr-profile=prog.profdata -format=lcov prog > prog.lcov
```

Subcommands: `show`, `report`, `export`, `gcov`.

### `llvm-profgen`

Generate SPGO (sampling PGO) profiles from Linux `perf` data:
```bash
llvm-profgen --binary=prog --perfdata=perf.data --output=prog.profdata
llvm-profgen --csspgo --binary=prog --perfdata=perf.data -o cs.profdata
```

---

## C.6 Analysis Tools

### `llvm-mca` (Machine Code Analyzer)

Throughput and latency analysis for assembly sequences:
```bash
llvm-mca -march=x86-64 -mcpu=skylake input.s
llvm-mca -march=aarch64 -mcpu=neoverse-v1 -timeline -iterations=100 input.s
```

Key flags: `--mcpu=<cpu>`, `--march=<arch>`, `--iterations=N`, `--timeline`, `--bottleneck-analysis`, `--all-views`, `--resource-pressure`, `--dispatch-stats`, `--output-asm-variant=0/1`.

Output sections: Instructions (throughput, latency, RThroughput), Resource pressure, Timeline.

### `llvm-stress`

Generate random IR modules for testing:
```bash
llvm-stress --size=100 --seed=42 -o random.ll
opt -S -p "default<O2>" random.ll | llc
```

### `llvm-reduce`

Delta-debug IR to a minimal reproducer:
```bash
llvm-reduce --test=check.sh input.ll -o reduced.ll
```

`check.sh` must exit 0 when the bug is present. `llvm-reduce` iteratively removes functions, blocks, instructions.

Flags: `--test-arg=<arg>`, `--keep-one-successor-condition`, `--delta-passes=<list>`.

---

## C.7 JIT Tools

### `lli`

Execute LLVM IR with JIT:
```bash
lli input.ll
lli -jit-kind=orc input.ll
lli --extra-object=extra.o input.ll arg1 arg2
```

Flags: `--jit-kind=mcjit/orc/orc-lazy`, `--extra-module=<file>`, `--extra-object=<file>`, `--extra-library=<name>`, `--remote-mcjit`.

### `llvm-jitlink`

ORC JIT-based linker for testing JITLink:
```bash
llvm-jitlink file.o
llvm-jitlink -entry=_main file1.o file2.o
llvm-jitlink -harness harness.o -test test.o
```

Flags: `--show-graphs`, `--show-sizes`, `--pre-link-passes=<pipeline>`, `--noexec`.

---

## C.8 Linker (LLD)

LLD is LLVM's linker. It supports multiple output formats via different frontend binaries:

| Binary | Platform | Usage |
|---|---|---|
| `ld.lld` | ELF (Linux) | `clang -fuse-ld=lld` or invoke directly |
| `lld-link` | COFF/PE (Windows) | MSVC-style: `lld-link /out:prog.exe main.obj` |
| `ld64.lld` | MachO (macOS) | Rarely invoked directly; via `clang` on macOS |
| `wasm-ld` | WebAssembly | `clang --target=wasm32-wasi -fuse-ld=lld` |

Key `ld.lld` flags:

| Flag | Purpose |
|---|---|
| `--lto=full/thin` | LTO mode |
| `--thinlto-cache-dir=<dir>` | ThinLTO cache |
| `--plugin-opt=<key>=<val>` | Pass plugin options (for gold-style) |
| `-e <entry>` | Entry point |
| `--gc-sections` | Remove unused sections |
| `--icf=all/safe` | Identical code folding |
| `--strip-debug` / `--strip-all` | Strip debug/all symbols |
| `--export-dynamic` | Export all dynamic symbols |
| `-rpath <path>` | Runtime library search path |
| `--dynamic-linker=<path>` | ELF interpreter |
| `--version-script=<file>` | Symbol version script |
| `--wrap=<sym>` | Symbol wrapping |
| `--build-id[=sha1/md5/uuid/hex]` | Generate build ID |
| `-z norelro` / `-z relro -z now` | RELRO hardening |
| `--Map=<file>` | Link map output |
| `-T <script>` | Linker script |
| `--reproduce=<file>` | Capture reproduction tarball |

---

## C.9 TableGen

### `llvm-tblgen`

| Backend Flag | Output |
|---|---|
| `--gen-emitter` | Instruction encoder |
| `--gen-disassembler` | Disassembler decode tables |
| `--gen-instr-info` | `*InstrInfo.inc` |
| `--gen-register-info` | `*RegisterInfo.inc` |
| `--gen-subtarget` | `*Subtarget.inc` |
| `--gen-asm-writer` | Asm printer tables |
| `--gen-asm-matcher` | Asm parser/matcher tables |
| `--gen-dag-isel` | SelectionDAG instruction selector |
| `--gen-fast-isel` | FastISel tables |
| `--gen-callingconv` | Calling convention tables |
| `--gen-intrinsic-enums` | Intrinsic enum definitions |
| `--gen-intrinsic-impl` | Intrinsic property tables |
| `--gen-attrs` | Attribute enum |
| `--gen-searchable-tables` | Searchable lookup tables |
| `--gen-global-isel` | GlobalISel tables |
| `--gen-global-isel-combiner` | GICombiner rules |
| `--print-records` | Dump all records (debug) |
| `--print-sets` | Dump sets |

```bash
llvm-tblgen --gen-instr-info -I llvm/include -I llvm/lib/Target/X86 \
  llvm/lib/Target/X86/X86.td -o X86GenInstrInfo.inc
```

Also: `clang-tblgen` (Clang-specific records), `mlir-tblgen` (ODS/OpDef backends).

---

## C.10 MLIR Tools

### `mlir-opt`

Run MLIR passes on `.mlir` files:
```bash
mlir-opt --passes="linalg-generalize-named-ops,convert-linalg-to-affine-loops,lower-affine,convert-scf-to-cf,convert-cf-to-llvm,finalize-memref-to-llvm,convert-func-to-llvm" input.mlir -o output.mlir

mlir-opt --mlir-print-debuginfo --mlir-print-ir-after-all input.mlir
mlir-opt --verify-diagnostics input.mlir    # for FileCheck-based tests
```

Key flags: `--mlir-print-op-on-diagnostic`, `--mlir-print-ir-after-failure`, `--mlir-disable-threading`, `--mlir-timing`, `--allow-unregistered-dialect`, `--split-input-file` (for test files with `// -----`).

### `mlir-translate`

Translate between MLIR and other representations:
```bash
mlir-translate --mlir-to-llvmir input.mlir -o output.ll
mlir-translate --mlir-to-spirv input.mlir -o output.spv
mlir-translate --import-llvm input.ll -o output.mlir
```

### `mlir-reduce`

Delta-debug failing MLIR IR: `mlir-reduce --test=check.sh input.mlir`

### `mlir-pdll`

Compile PDLL pattern files to PDL/C++:
```bash
mlir-pdll patterns.pdll -x mlir -o patterns.mlir
mlir-pdll patterns.pdll -x cpp -o patterns.h
```

### `mlir-lsp-server`

Language server for MLIR `.mlir` files in VS Code/nvim. Run: `mlir-lsp-server` (the editor handles the LSP protocol).

---

## C.11 Clang Tools

### `clang-format`

```bash
clang-format -i *.cpp              # in-place format
clang-format --style=LLVM file.cpp # use LLVM style
clang-format --dump-config         # dump effective config
clang-format --style='{BasedOnStyle: LLVM, IndentWidth: 4}'
```

Style options: `LLVM`, `Google`, `Chromium`, `Mozilla`, `WebKit`, `Microsoft`, `GNU`, or `file` (read `.clang-format`).

### `clang-tidy`

```bash
clang-tidy -checks='clang-analyzer-*,modernize-*' file.cpp -- -std=c++20
clang-tidy --fix --fix-errors file.cpp --
clang-tidy --list-checks | grep modernize
clang-tidy -export-fixes=fixes.yaml file.cpp --
clang-apply-replacements fixes/             # apply saved fixes
```

Key check prefixes: `clang-analyzer-*`, `modernize-*`, `performance-*`, `readability-*`, `bugprone-*`, `cppcoreguidelines-*`, `misc-*`, `cert-*`, `google-*`, `abseil-*`.

### `clangd`

LSP server for C++. Typically launched by editor plugin. Key config (`.clangd`):
```yaml
CompileFlags:
  Add: [-std=c++20, -Wall]
  Remove: [-W*]
Diagnostics:
  ClangTidy:
    Add: [modernize-*, performance-*]
    Remove: [modernize-use-trailing-return-type]
```

### `scan-build` / `scan-view`

```bash
scan-build -o /tmp/scan-results make
scan-view /tmp/scan-results/2024-01-01-*
```

### `clang-query`

Interactive AST query tool:
```bash
clang-query file.cpp -- -std=c++20
# then: match functionDecl(hasName("foo"))
# or:   match callExpr(callee(functionDecl(hasName("malloc"))))
```

### `clang-refactor`

Extract/rename source transformations:
```bash
clang-refactor rename -new-name=bar --qualified-name=foo file.cpp --
clang-refactor extract -extract-function=helper -selection=file.cpp:10:1-15:1 file.cpp --
```

---

## C.12 Debugging: LLDB

```bash
lldb prog                    # start debugger
lldb -- prog arg1 arg2       # with args
lldb -p <pid>                # attach to process
lldb prog -core core.dump    # analyze core dump
```

Key LLDB commands:

| Command | Action |
|---|---|
| `b main` / `b file.cpp:42` | Set breakpoint |
| `rb foo` | Set breakpoint by regex on function name |
| `r` / `run [args]` | Run |
| `c` / `continue` | Continue |
| `n` / `next` | Step over |
| `s` / `step` | Step into |
| `fin` / `finish` | Step out |
| `p expr` / `po obj` | Print expression / object description |
| `fr v` | Print frame variables |
| `bt` | Backtrace |
| `up` / `down` | Navigate frames |
| `dis -f` | Disassemble current function |
| `mem read -s4 -fu 0x400000` | Read memory as uint32 |
| `watch set var foo` | Watchpoint |
| `expr --language c++ -- (int)foo()` | Evaluate expression |
| `settings set target.max-children-count 100` | Display settings |

---

## C.13 Sanitizer Environment Variables

### `ASAN_OPTIONS`

| Option | Default | Description |
|---|---|---|
| `abort_on_error` | 0 | Abort instead of exit on error |
| `detect_stack_use_after_return` | 0 | Detect use-after-return (slow) |
| `detect_leaks` | 1 | Enable leak detection |
| `fast_unwind_on_malloc` | 1 | Fast unwind in malloc hooks |
| `halt_on_error` | 1 | Stop on first error |
| `log_path` | stderr | Log file path |
| `malloc_context_size` | 30 | Stack depth for malloc reports |
| `max_malloc_fill_size` | 0 | Fill allocated memory with pattern |
| `print_stats` | 0 | Print memory stats on exit |
| `quarantine_size_mb` | 256 | Quarantine buffer size |
| `redzone` | 16 | Redzone size in bytes |
| `symbolize` | 1 | Symbolize addresses |
| `suppressions` | "" | Suppression file path |
| `verbosity` | 0 | Verbosity level |

### `TSAN_OPTIONS`

| Option | Description |
|---|---|
| `halt_on_error` | Stop on first race |
| `report_atomic_races` | Report races on atomics |
| `second_deadlock_stack` | Show second stack in deadlock |
| `suppressions` | Suppression file |
| `history_size` | Race history size (0–7) |
| `io_sync` | Synchronize I/O (0=none, 1=write, 2=all) |

### `MSAN_OPTIONS`

| Option | Description |
|---|---|
| `poison_in_dtor` | Poison memory in destructors |
| `print_stats` | Print memory stats |
| `wrap_signals` | Intercept signal delivery |
| `track_origins` | Track uninitialized memory origins (0-2, slower) |

### `UBSAN_OPTIONS`

| Option | Description |
|---|---|
| `print_stacktrace` | Print stack on error |
| `halt_on_error` | Stop on first error |
| `suppressions` | Suppression file |
| `report_error_type` | Include error type in report |

General format: `ASAN_OPTIONS="key1=val1:key2=val2" ./prog`


---

@copyright jreuben11
