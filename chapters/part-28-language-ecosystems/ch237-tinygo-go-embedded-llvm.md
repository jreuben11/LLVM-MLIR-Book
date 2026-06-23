# Chapter 237 — TinyGo: Compiling Go for Embedded Systems and WebAssembly via LLVM

*Part XXVIII — Language Ecosystems and Alternative Front-Ends*

The Go toolchain from `golang.org/dl` compiles to a garbage-collected, goroutine-scheduled runtime that requires at least several hundred kilobytes of memory and a functioning OS. This makes it unsuitable for microcontrollers with 32 KB of flash and 4 KB of RAM. TinyGo addresses this constraint by replacing the entire Go compilation pipeline with a new compiler that emits LLVM IR, allowing Go programs to target bare-metal ARM Cortex-M, RISC-V, AVR, and WebAssembly with a fraction of the original footprint. This chapter traces TinyGo's architecture from the `go/ssa` frontend through its LLVM IR emitter, runtime specialization, and target description format.

---

## Table of Contents

- [237.1 Design Goals and the go/ssa Frontend](#2371-design-goals-and-the-gossa-frontend)
- [237.2 Compilation Pipeline: compiler.go to LLVM IR](#2372-compilation-pipeline-compilergo-to-llvm-ir)
- [237.3 The go-llvm Wrapper and Custom DataLayout](#2373-the-go-llvm-wrapper-and-custom-datalayout)
  - [Custom DataLayout per target](#custom-datalayout-per-target)
- [237.4 Garbage Collector Options](#2374-garbage-collector-options)
  - [Conservative GC internals](#conservative-gc-internals)
  - [Precise GC internals](#precise-gc-internals)
- [237.5 Goroutines as LLVM Coroutines](#2375-goroutines-as-llvm-coroutines)
  - [Transformation pipeline](#transformation-pipeline)
- [237.6 Interface Lowering](#2376-interface-lowering)
- [237.7 Target JSON Files](#2377-target-json-files)
- [237.8 Standard Library Subset, go:linkname, and FFI](#2378-standard-library-subset-golinkname-and-ffi)
  - [stdlib subset](#stdlib-subset)
  - [go:linkname for runtime intrinsics](#golinkname-for-runtime-intrinsics)
  - [Exporting Go functions to C](#exporting-go-functions-to-c)
  - [wasm_exec.js and WASI](#wasmexecjs-and-wasi)
- [Summary](#summary)

---

## 237.1 Design Goals and the go/ssa Frontend

TinyGo is not a re-implementation of the Go specification from scratch. It reuses two major components of the standard Go toolchain:

1. **`golang.org/x/tools/go/packages`** — package loading, type-checking, and dependency resolution. This ensures TinyGo processes the same import graph as `go build`.
2. **`golang.org/x/tools/go/ssa`** — Static Single Assignment form intermediate representation. The `go/ssa` package converts Go's AST into an SSA IR that TinyGo then lowers to LLVM IR.

The decision to use `go/ssa` rather than work from the AST directly is significant: `go/ssa` performs type erasure, interface flattening setup, and control-flow normalization before TinyGo sees the program. This means TinyGo inherits the correct semantics for complex features (closures, deferred calls, type assertions) from the upstream SSA builder without reimplementing them.

**vs. the `gc` toolchain**: The standard `gc` compiler (`cmd/compile`) generates Go's own IR (`.a` archive files), runs SSA optimizations in `cmd/compile/internal/ssa`, and then calls the Go assembler. TinyGo's path diverges immediately after package loading — it calls `go/ssa.NewProgram` and never touches `cmd/compile`.

**Package `tinygo.org/x/go-llvm`**: TinyGo's LLVM C API wrapper is maintained as a separate module at `github.com/tinygo-org/tinygo/compiler/` with the `go-llvm` package providing Go bindings. It wraps `libLLVM.so` via CGo.

---

## 237.2 Compilation Pipeline: compiler.go to LLVM IR

The entry point is `compiler/compiler.go`. The `Compiler` struct holds the LLVM `Context`, `Module`, and `Builder`:

```go
// compiler/compiler.go (simplified)
type Compiler struct {
    mod           llvm.Module
    ctx           llvm.Context
    builder       llvm.Builder
    program       *ssa.Program
    machine       llvm.TargetMachine
    targetData    llvm.TargetData
    config        *compileopts.Config
    funcImpls     map[*ssa.Function]llvm.Value
    globalVars    map[*ssa.Global]llvm.Value
}
```

Compilation proceeds in phases:

**Phase 1 — SSA construction**:
```go
prog := ssa.NewProgram(fset, ssa.SanityCheckFunctions)
for _, pkg := range pkgs {
    prog.CreatePackage(pkg)
}
prog.Build()
```

**Phase 2 — Global declarations**: All package-level variables and function signatures are declared in the LLVM module before any function bodies are emitted. This handles forward references.

**Phase 3 — Function body emission**: Each `*ssa.Function` is lowered by `c.parseFunc(f)`. The translator walks SSA instructions in block order:

```go
func (c *Compiler) parseInstr(frame *Frame, instr ssa.Instruction) {
    switch instr := instr.(type) {
    case *ssa.Alloc:
        c.emitAlloc(frame, instr)
    case *ssa.BinOp:
        c.emitBinOp(frame, instr)
    case *ssa.Call:
        c.emitCall(frame, instr)
    case *ssa.MakeInterface:
        c.emitMakeInterface(frame, instr)
    // ... 40+ cases
    }
}
```

**Phase 4 — Optimizations**: After IR emission, TinyGo runs a custom optimization pipeline via `OptimizeModule`:

```go
// compiler/optimizer.go
func OptimizeModule(mod llvm.Module, config *compileopts.Config, sizeLevel int) {
    pm := llvm.NewPassManager()
    defer pm.Dispose()
    // GlobalDCE first (Go generates many unreachable declarations)
    pm.AddGlobalDCEPass()
    // Then standard -O2 or -Oz pipeline depending on sizeLevel
    pmb := llvm.NewPassManagerBuilder()
    pmb.SetOptLevel(config.OptLevel())
    pmb.SetSizeLevel(sizeLevel)
    pmb.PopulateModulePassManager(pm)
    pm.Run(mod)
}
```

**Phase 5 — Code generation**: `machine.EmitToMemoryBuffer(mod, llvm.ObjectFile)` produces an object file, followed by LLD linking.

---

## 237.3 The go-llvm Wrapper and Custom DataLayout

TinyGo's Go–LLVM binding (`go-llvm`) is a CGo wrapper around `libLLVM`. It is not the same as the `llvm.org/llvm/bindings/go` package (which was removed from the LLVM monorepo). TinyGo maintains it as a vendored package with Android, macOS, Linux, and Windows prebuilt `libLLVM` artifacts.

### Custom DataLayout per target

Each TinyGo target defines an explicit data layout string that overrides the defaults in the LLVM target:

```go
// For thumbv7em (Cortex-M4)
"e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64"
```

Breaking this down:
- `e` — little-endian
- `m:e` — ELF mangling
- `p:32:32` — 32-bit pointers, 32-bit aligned
- `Fi8` — function pointer alignment 8 bits (important for Thumb instruction set)
- `i64:64` — 64-bit integers 64-bit aligned
- `v128:64:128` — 128-bit vectors 64-bit aligned, 128-bit preferred
- `a:0:32` — aggregate alignment 0 ABI, 32 preferred
- `n32` — native integer width 32 bits (for SelectionDAG width hints)
- `S64` — stack alignment 64 bits (8-byte stack frames)

The data layout string is passed to `llvm.NewTargetMachineFromTriple` and then stored in the module with `mod.SetDataLayout(dl)`. Mismatches between the data layout and the actual target produce incorrect code silently — TinyGo enforces this via target JSON files (§237.7).

---

## 237.4 Garbage Collector Options

TinyGo supports four GC strategies selectable at compile time with `-gc=<strategy>`:

| Strategy | Flag | Description |
|----------|------|-------------|
| None | `-gc=none` | No GC; allocations never freed. Suitable for programs with bounded lifetimes (event loops, one-shot firmware). |
| Leaking | `-gc=leaking` | Synonym for none; explicit name makes intent clear |
| Conservative | `-gc=conservative` | Stop-the-world mark-and-sweep scanning all words. No write barriers. Works with C interop. Default for most bare-metal targets. |
| Precise | `-gc=precise` | Precise mark-and-sweep using type metadata. Smaller heap fragmentation. Requires GC metadata emitted during compilation. Default for Linux/macOS/wasm. |

### Conservative GC internals

The conservative collector scans the entire Go stack and all global variables as potential roots. Any word-sized value that looks like a heap pointer (falls within the heap address range) is treated as a live pointer. This means:

- An integer that happens to equal a heap address pins that object
- No write barriers → simpler IR, lower overhead
- Works correctly with CGo opaque pointers

The collector is implemented in `src/runtime/gc_conservative.go`. The `alloc(size uintptr)` runtime function calls into the GC when the heap runs out:

```go
// runtime/gc_conservative.go
func alloc(size uintptr, layout unsafe.Pointer) unsafe.Pointer {
    if heapEnd+size > heapMax {
        GC()
    }
    ptr := heapEnd
    heapEnd += align(size, 4)
    return ptr
}
```

### Precise GC internals

The precise GC uses pointer bitmaps emitted alongside each type's allocation site. TinyGo's compiler emits `runtime.typeBitmap` metadata for every heap-allocated type. The collector follows only genuine pointer fields, reducing false pinning. This is critical on wasm where 32-bit addresses are dense and false positives are frequent.

---

## 237.5 Goroutines as LLVM Coroutines

The `go` statement creates a goroutine. In TinyGo, goroutines are **not** OS threads. Instead, TinyGo uses **cooperative scheduling** backed by LLVM's coroutine intrinsics (`llvm.coro.*`).

### Transformation pipeline

1. **Goroutine detection**: Functions that may be suspended (blocked on channel, `time.Sleep`, `sync.Mutex`) are identified via a fixed-point reachability analysis.

2. **Coroutine splitting**: Suspendable functions are transformed using LLVM's `CoroSplit` pass. The transformation inserts `llvm.coro.id`, `llvm.coro.begin`, `llvm.coro.suspend`, `llvm.coro.end`:

```llvm
define void @goroutine_body(%coro.frame* %frame) {
entry:
  %id  = call token @llvm.coro.id(i32 0, i8* null, i8* null, i8* null)
  %hdl = call i8* @llvm.coro.begin(token %id, i8* null)
  ; ... setup work ...
  %sp  = call i8 @llvm.coro.suspend(token none, i1 false)
  switch i8 %sp, label %suspend [
    i8 0, label %resume
    i8 1, label %cleanup
  ]
resume:
  ; ... resumed work ...
  br label %cleanup
cleanup:
  call i1 @llvm.coro.end(i8* %hdl, i1 false)
  ret void
}
```

3. **Scheduler**: The TinyGo runtime maintains a queue of runnable coroutine handles (`src/runtime/scheduler_cooperative.go`). The main loop calls `resume(handle)` for each runnable goroutine:

```go
// runtime/scheduler_cooperative.go
func scheduler() {
    for {
        if head := runqueue.pop(); head != nil {
            head.resume()  // calls into LLVM coroutine resume function
        } else if timerQueue.peek() != nil {
            waitForInterrupt()
        } else {
            return  // all goroutines done
        }
    }
}
```

This model requires no RTOS or threading primitives — the entire execution is single-threaded from the OS/hardware perspective. It works on bare-metal Cortex-M with no OS at all.

**Limitation**: Blocking C calls (`CGo`) block the entire scheduler. TinyGo advises wrapping blocking C calls in a dedicated goroutine with a channel bridge, or using async-style callbacks.

**Channels**: Channel send/receive are implemented as yield points. A blocked goroutine suspends via `llvm.coro.suspend` and is re-enqueued by the goroutine that unblocks it (sender puts the value and resumes the waiting receiver).

---

## 237.6 Interface Lowering

Go interfaces are pairs of (dynamic type, value). TinyGo lowers interfaces to a two-word struct:

```go
// Internal representation:
type _interface struct {
    typeID uint16        // index into the global type table
    value  unsafe.Pointer // pointer to value (or scalar inline if fits)
}
```

Using `uint16` for the type ID is a TinyGo-specific optimization: programs targeting small MCUs rarely have more than 65535 interface types. This halves the interface header size compared to Go's `itab *` pointer (which is 8 bytes on 64-bit).

**Method dispatch**: TinyGo emits a `switch` over `typeID` at interface call sites rather than a vtable pointer dereference. For large programs with many types implementing an interface this produces larger code but avoids indirect calls (important for CFI and for CPUs without branch predictors). The threshold is configurable; at `-opt=z` TinyGo switches to smaller vtable-based dispatch.

```llvm
; Interface call: x.String() where x has typeID
switch i16 %typeid, label %panic [
  i16 1, label %call_string_myType
  i16 7, label %call_string_otherType
  ; ...
]
call_string_myType:
  %result = call %string @myType.String(%ptr)
  br label %merge
```

**Type assertions**: `x.(T)` is a comparison of `typeID` against T's compile-time-assigned ID, followed by a pointer cast. No `reflect.Type` lookup is needed.

---

## 237.7 Target JSON Files

TinyGo describes each supported target with a JSON file in `targets/`. These files specify the LLVM triple, CPU, features, data layout, linker flags, and memory map:

```json
// targets/cortex-m4.json
{
  "inherits": ["arm"],
  "cpu": "cortex-m4",
  "triple": "thumbv7em-unknown-unknown-eabi",
  "features": "+armv7e-m,+fp16,+neon,+soft-float,+thumb-mode,+vfp4d16sp,-fp64",
  "datalayout": "e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64",
  "scheduler": "tasks",
  "linkerscript": "targets/cortex-m.ld",
  "extra-files": [
    "src/device/arm/arm.s",
    "src/runtime/interrupt/interrupt_cortexm.go"
  ],
  "flash-method": "openocd",
  "openocd-interface": "cmsis-dap",
  "openocd-target": "stm32f4x"
}
```

Supported target families:

| Family | Triple | CPU examples |
|--------|--------|--------------|
| ARM Cortex-M0/M0+ | `thumbv6m-unknown-unknown-eabi` | `cortex-m0`, `cortex-m0plus` |
| ARM Cortex-M4/M7 | `thumbv7em-unknown-unknown-eabihf` | `cortex-m4`, `cortex-m7` |
| ARM Cortex-M33/M55 | `thumbv8m.main-unknown-unknown-eabihf` | `cortex-m33` |
| RISC-V 32 | `riscv32-unknown-unknown` | `generic-rv32`, `qingke-v4b` |
| AVR | `avr-unknown-unknown` | `atmega328p`, `avr5` |
| wasm (WASI) | `wasm32-unknown-wasi` | generic |
| wasm (browser) | `wasm32-unknown-unknown` | generic |
| Linux/amd64 | `x86_64-unknown-linux-musl` | `x86-64` |

Target JSON files can `"inherits"` from a parent to reduce repetition. Board-specific files (e.g., `targets/arduino-nano.json`) inherit from `targets/avr.json` and add only the board-specific flash method and pin definitions.

---

## 237.8 Standard Library Subset, go:linkname, and FFI

### stdlib subset

Not all of the Go standard library is available in TinyGo. TinyGo ships its own implementations of commonly used packages in `src/`:

| Package | Status |
|---------|--------|
| `fmt` | Reduced (no `%v` reflection) |
| `os` | Partial (file I/O on WASI/Linux; stub on bare-metal) |
| `sync` | Full (mutexes map to scheduler primitives) |
| `math` | Full (maps to LLVM `llvm.sqrt`, `llvm.sin`, etc.) |
| `reflect` | Minimal (type name, kind; no `reflect.Value.Set`) |
| `net/http` | Not available on bare-metal; available with network stack |
| `database/sql` | Not available |

TinyGo patches missing stdlib packages with `//go:build tinygo` build tags that select alternative implementations.

### go:linkname for runtime intrinsics

TinyGo uses `//go:linkname` extensively to wire up runtime functions implemented in assembly or C. For example, the `runtime.memcpy` used in slice operations is linked to `__aeabi_memcpy4` on ARM or `memcpy` on wasm:

```go
// src/runtime/mem.go
//go:linkname memcpy runtime.memcpy
func memcpy(dst, src unsafe.Pointer, n uintptr) {
    // implemented in assembly for each target
}
```

### Exporting Go functions to C

The `//export` directive creates a C-visible symbol:

```go
//export add
func add(a, b int32) int32 {
    return a + b
}
```

This emits:
```llvm
define i32 @add(i32 %a, i32 %b) #0 {
  %result = add i32 %a, %b
  ret i32 %result
}
```

With the `dllexport` attribute on Windows and standard ELF export on Linux/wasm.

### wasm_exec.js and WASI

For browser wasm targets, TinyGo provides a thin JavaScript shim (`wasm_exec.js`) similar to Go's own but smaller. It implements `syscall/js` bindings and wires up the Go memory model to `WebAssembly.Memory`:

```javascript
// wasm_exec.js (TinyGo variant)
const go = new TinyGo();
WebAssembly.instantiateStreaming(fetch("program.wasm"), go.importObject)
  .then(result => go.run(result.instance));
```

For WASI targets (`wasm32-unknown-wasi`), TinyGo generates a `_start` function that calls `runtime.run()` and uses WASI system calls via `wasi_snapshot_preview1` imports. No JavaScript shim is needed — the `.wasm` file runs in any WASI runtime (Wasmtime, WasmEdge, Wasmer).

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **New LLVM pass manager migration**: TinyGo's `OptimizeModule` still uses the legacy `PassManager` API via `go-llvm`. Upstream tracking issue [tinygo-org/tinygo#3755](https://github.com/tinygo-org/tinygo/issues/3755) covers migrating to the new `PassBuilder` / `llvm::PassManager<Module>` C API introduced in LLVM 14 and stabilized in LLVM 16–22. This unlocks use of `llvm::ModuleInlinerWrapperPass` and the `ThinLTO` summary pipeline.
- **ARM Cortex-M55 and Helium (MVE) target support**: The `thumbv8.1m.main` triple and MVE (M-Profile Vector Extension) intrinsics are underrepresented; community PRs for the Renesas RA6 series (Cortex-M33) and STMicro STM32H5 (Cortex-M33+TrustZone) are in review. LLVM 22's improved `ARMTargetMachine` MVE scheduler model enables better auto-vectorization for DSP workloads.
- **WASI Preview 2 / Component Model**: The `wasm32-unknown-wasi` target currently targets `wasi_snapshot_preview1`. The WASI component model (Preview 2, stabilized in Wasmtime 18 and WasmEdge 0.14) requires a new `wasm32-wasip2` triple and WIT-generated glue; TinyGo RFC discussion is active on the tinygo-org Slack `#wasm` channel.
- **Precise GC metadata format convergence**: The TinyGo precise GC's `runtime.typeBitmap` format is hand-maintained. Ongoing work aligns it with the LLVM `gc.statepoint` / `gc.relocate` infrastructure so that the LLVM GC plugin API (`llvm::GCStrategy`) can manage root enumeration, reducing hand-rolled stack scanning code in `src/runtime/gc_precise.go`.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **RISC-V 64 and RV32E bare-metal targets**: RISC-V 64-bit (`riscv64-unknown-unknown`) support for boards like the SiFive U74 and ESP32-P4 is prototyped but not production-ready. The LLVM RISC-V backend improvements in LLVM 18–22 (Zicond, Zba/Zbb/Zbc bit-manipulation, vector "V" extension) create a foundation for TinyGo to emit RVV (RISC-V Vector) instructions for signal processing on embedded Linux SoCs.
- **Preemptive goroutine scheduling via signals**: The current cooperative coroutine scheduler (`llvm.coro.*`) cannot preempt compute-bound goroutines. Academic prototypes (cf. Behrends et al., "Preemptive Concurrency in Resource-Constrained Runtimes", LCTES 2024) demonstrate inserting preemption checks using LLVM safepoint polls (`llvm.experimental.gc.statepoint`) at loop back-edges; integration into TinyGo would require backend safepoint insertion and context-save/restore for MCU register banks.
- **`go-llvm` wrapper replacement with `llvm-sys` / CGo 2.0**: TinyGo's `go-llvm` CGo wrapper incurs significant cgo call overhead for each LLVM API call during IR emission. A proposed redesign would batch IR emission via a protobuf or FlatBuffers wire format to a separate `tinygo-llvm-worker` process, similar to how `rustc` uses `librustc_llvm`. This would also decouple TinyGo's LLVM version from the host's installed `libLLVM.so`.
- **Full `reflect` package support**: The current minimal `reflect` implementation blocks use of JSON encoding/decoding, gRPC-generated stubs, and protocol buffers. TinyGo developers are building a compile-time reflection table emitter that encodes type layouts in a read-only `.rodata` section, enabling `reflect.Type` / `reflect.Value` without a heap-resident type registry.

### 5-Year Horizon (Long-Term, by ~2031)

- **Integration with MLIR's `emitc` dialect for microcontroller retargetability**: As MLIR's `emitc` dialect matures (targeting C/C++ textual output for MCUs that lack a qualified LLVM backend, such as certain 8-bit PICs and proprietary DSPs), TinyGo could lower `go/ssa` to MLIR and use `emitc` for targets where full LLVM codegen is impractical, enabling Go programs on a much broader class of embedded systems.
- **Hardware-assisted memory safety via Arm Memory Tagging Extension (MTE)**: Arm Cortex-A510/A710 and future Cortex-M85+ cores support MTE (FEAT_MTE2/MTE3). LLVM's MTE instrumentation pass (`llvm::MemTagSanitizer`) could be adapted for TinyGo's precise GC to tag heap allocations at runtime, providing spatial and temporal safety guarantees previously available only in Rust-style ownership systems for embedded targets.
- **Go module ecosystem on WASM Components**: The Bytecode Alliance's WebAssembly Component Model roadmap projects production-grade language-agnostic component linking by 2028–2029. TinyGo, as a natural fit for compact WASM components, would need to generate WIT-typed exports, support multi-memory proposals (for shared-nothing isolation), and interoperate with Wit-bindgen-generated C/Rust glue — creating a viable path for Go libraries as universal WASM components deployable on cloud edge runtimes (Fastly Compute, Cloudflare Workers).

---

## Summary

- TinyGo reuses `golang.org/x/tools/go/ssa` for frontend parsing and SSA construction, diverging from `cmd/compile` at the IR level.
- `compiler/compiler.go` walks `*ssa.Function` bodies and emits LLVM IR via the `go-llvm` CGo wrapper around `libLLVM.so`.
- Custom DataLayout strings per target ensure correct ABI for 32-bit MCU address spaces, Thumb function pointer alignment, and vector register alignment.
- Four GC modes (`none`, `leaking`, `conservative`, `precise`) trade correctness for code size and simplicity.
- Goroutines are cooperative coroutines via LLVM `llvm.coro.*` intrinsics and `CoroSplit`; the single-threaded scheduler requires no RTOS.
- Interfaces are two-word `{uint16 typeID, ptr}` pairs; dispatch uses a `switch` over typeIDs rather than vtable pointers.
- Target JSON files describe triple, CPU, features, memory layout, linker script, and flash method; they cover Cortex-M0/M4/M7/M33, RV32, AVR, wasm/WASI, and Linux/musl.
- The stdlib subset is governed by `//go:build tinygo` tags; `//export` and `//go:linkname` bridge Go to C and assembly; WASI targets need no JS shim.
