# Chapter 116 — LLDB Architecture

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

LLDB is LLVM's debugger: a modern, modular, library-first replacement for GDB that shares the LLVM infrastructure for disassembly, debug info parsing, type systems, and expression evaluation. While GDB is a monolithic tool with a decades-long history of accretion, LLDB was designed from the ground up with a layered plugin architecture, an embedded Clang compiler for live expression evaluation, and a clean separation between the debugger library (`liblldb`) and the user interface. This chapter covers LLDB's architecture in detail: its core abstractions, plugin system, expression evaluator pipeline, MC integration, symbol resolution, and remote debugging over the GDB Remote Serial Protocol.

---

## 116.1 LLDB Overview

### Design Philosophy

LLDB's fundamental design principle is that debugger functionality should be exposed as a reusable library (`liblldb`), not buried in a command-line tool. This enables IDE integration (Xcode's debugger, CLion, VS Code via `lldb-dap`), language-server-style debugger backends, and programmatic debugging via the Python scripting interface.

Three key differentiators from GDB:

1. **Embedded Clang**: LLDB can parse and compile C, C++, Objective-C, and Swift expressions in the context of the debugged process. The expression evaluator is not a simple interpreter but a full compilation pipeline that generates JIT-compiled machine code.

2. **LLVM infrastructure reuse**: LLDB uses `MCDisassembler` for disassembly, `DWARFContext` for debug info, LLVM's type system, and ORC JIT for expression evaluation — avoiding reimplementation of complex subsystems.

3. **Remote-first architecture**: all debugging operations, including local debugging, use the same `ProcessGDBRemote` interface internally. The `lldb-server` process handles actual process control; `lldb` is a client.

### Platform Support

LLDB runs on macOS, Linux, Windows (MSVC targets), FreeBSD, and Android. It supports remote debugging over GDB Remote Serial Protocol (RSP) and custom protocol extensions. The `lldb-server` binary provides both a GDB-compatible server (for IDE integration) and a LLDB-native platform server (for remote file transfer and process management).

### Source Tree Organization

```
lldb/
├── include/lldb/            # Public headers
│   ├── API/                 # SB (ScriptBridge) API headers
│   └── Core/, Target/, ...  # Internal headers
├── source/
│   ├── API/                 # SB API implementation
│   ├── Core/                # Core abstractions (Debugger, etc.)
│   ├── Commands/            # Command implementations
│   ├── Expression/          # Expression evaluator
│   ├── Host/                # OS abstraction layer
│   ├── Interpreter/         # Command interpreter
│   ├── Plugins/             # All plugin implementations
│   │   ├── ABI/
│   │   ├── Disassembler/
│   │   ├── DynamicLoader/
│   │   ├── ExpressionParser/
│   │   ├── Language/
│   │   ├── ObjectFile/
│   │   ├── Platform/
│   │   ├── Process/
│   │   ├── SymbolFile/
│   │   └── ...
│   ├── Symbol/              # Symbol table, type system
│   └── Target/              # Target, Thread, Frame, Breakpoint
└── tools/
    ├── lldb/                # lldb command-line tool
    ├── lldb-server/         # lldb-server (gdbserver + platform)
    └── lldb-dap/            # DAP adapter (for VS Code etc.)
```

---

## 116.2 Core Abstractions

LLDB's object model is a strict hierarchy:

```
Debugger
  │
  ├── Target (1 or more)
  │     │
  │     ├── Module list (executable + loaded shared libraries)
  │     │     └── Module
  │     │           ├── ObjectFile (ELF/Mach-O/PE/COFF)
  │     │           └── SymbolFile (DWARF/PDB/dSYM)
  │     │
  │     ├── Breakpoint list
  │     │
  │     └── Process
  │           │
  │           ├── Thread list
  │           │     └── Thread
  │           │           └── StackFrame list (frame 0 = innermost)
  │           │                 └── StackFrame
  │           │                       ├── SymbolContext (function, block, location)
  │           │                       ├── VariableList (locals, args)
  │           │                       └── RegisterContext
  │           │
  │           └── Memory access API
  │
  └── Listener (event dispatch)
```

### Debugger

`lldb_private::Debugger` ([`lldb/source/Core/Debugger.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lldb/source/Core/Debugger.cpp)) is the root object. Each `Debugger` instance has:

- A list of `Target` objects
- A `CommandInterpreter` for command parsing
- An event dispatch system (`Broadcaster`/`Listener`)
- I/O handlers for terminal interaction

Multiple `Debugger` instances can coexist in the same process (useful for embedding LLDB in IDEs that manage multiple debug sessions).

### Target

`lldb_private::Target` represents a debuggable program:

```cpp
// Creating a target (SB API):
lldb::SBTarget target = debugger.CreateTarget("/path/to/binary");

// Or with platform:
lldb::SBTarget target = debugger.CreateTargetWithFileAndArch(
    "/path/to/binary", "x86_64");
```

A Target exists before any process is started (useful for examining a binary, setting breakpoints by symbol name, etc.). Breakpoints set on a Target are preserved across process restarts.

### Process

`lldb_private::Process` represents a running (or stopped) process. Operations include:

```cpp
// SB API:
lldb::SBLaunchInfo launch_info(nullptr);
launch_info.SetArguments(argv, false);
lldb::SBError error;
lldb::SBProcess process = target.Launch(launch_info, error);

// Read memory:
uint8_t buf[16];
size_t bytes_read = process.ReadMemory(addr, buf, sizeof(buf), error);

// Continue execution:
process.Continue();
// or step:
thread.StepInstruction(/*step_over=*/false);
```

### Thread and StackFrame

```cpp
// Iterate threads:
for (uint32_t i = 0; i < process.GetNumThreads(); i++) {
    lldb::SBThread thread = process.GetThreadAtIndex(i);
    
    // Get stack:
    for (uint32_t f = 0; f < thread.GetNumFrames(); f++) {
        lldb::SBFrame frame = thread.GetFrameAtIndex(f);
        
        // Source location:
        lldb::SBLineEntry line_entry = frame.GetLineEntry();
        printf("#%d  %s at %s:%d\n",
               f, frame.GetFunctionName(),
               line_entry.GetFileSpec().GetFilename(),
               line_entry.GetLine());
        
        // Variables:
        lldb::SBValueList vars = frame.GetVariables(
            /*args=*/true, /*locals=*/true, /*statics=*/false, /*in_scope=*/true);
        for (uint32_t v = 0; v < vars.GetSize(); v++) {
            lldb::SBValue var = vars.GetValueAtIndex(v);
            printf("  %s = %s\n", var.GetName(), var.GetValue());
        }
    }
}
```

---

## 116.3 Plugin Architecture

LLDB's plugin system allows every non-core capability to be provided by interchangeable implementations. The plugin manager maintains a registry of plugins by type and name:

```cpp
// Plugin registration (happens at static init time):
// lldb/source/Plugins/ObjectFile/ELF/ObjectFileELF.cpp
void ObjectFileELF::Initialize() {
    PluginManager::RegisterPlugin(GetPluginNameStatic(),
                                   GetPluginDescriptionStatic(),
                                   CreateInstance,
                                   CreateMemoryInstance,
                                   GetModuleSpecifications);
}
```

### Plugin Categories

| Category | Examples | Purpose |
|----------|---------|---------|
| `ObjectFile` | ELF, MachO, PECOFF, wasm | Parse binary formats |
| `SymbolFile` | DWARF, PDB, Breakpad, compact-unwind | Parse debug info |
| `Platform` | Linux, macOS, Android, iOS, Windows | OS-specific operations |
| `Process` | GDBRemote, macOS (mach), Windows (native) | Process control |
| `ABI` | SysV_x86_64, ARM_EABI, AArch64 | Calling convention, return value extraction |
| `Disassembler` | LLVM_MC (wraps LLVM's MCDisassembler) | Instruction disassembly |
| `Language` | C++, ObjC, Swift, Rust | Language-specific expression syntax |
| `DynamicLoader` | POSIX-DYLD, Darwin-DYLD, LLDB-Process-DYLD | `dlopen` tracking |
| `JITLoader` | GDB (JIT interface), PERF_JITDUMP | JIT symbol notification |
| `Unwind` | eh_frame, compact-unwind, Assembly | Stack unwinding |

### Selecting a Plugin

When LLDB needs to create a `SymbolFile` for a loaded module, it tries each registered plugin in priority order:

```cpp
// lldb/source/Symbol/SymbolFile.cpp:
SymbolFile *SymbolFile::FindPlugin(ObjectFileSP object_file_sp) {
    SymbolFileCreateInstance create_callback;
    for (uint32_t idx = 0;
         (create_callback = PluginManager::GetSymbolFileCreateCallbackAtIndex(idx)) != nullptr;
         idx++) {
        std::unique_ptr<SymbolFile> instance(create_callback(object_file_sp, false));
        if (instance)
            return instance.release();
    }
    return nullptr;
}
```

The plugin that claims the highest "ability" score for the given binary wins.

---

## 116.4 Expression Evaluation

LLDB's expression evaluator is one of its most powerful and unique features. When you type `p my_object.method()` in the LLDB REPL, LLDB:

1. Parses the expression string with an embedded Clang instance.
2. Generates LLVM IR for the expression in the context of the current frame.
3. JIT-compiles the IR to machine code.
4. Allocates memory in the target process and copies the machine code there.
5. Calls the machine code in the target process.
6. Reads the return value back from the target process.

This is far more powerful than GDB's expression evaluator, which is a custom interpreter that handles only a subset of C/C++ expressions.

### The Compilation Pipeline

[`lldb/source/Expression/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lldb/source/Expression/) contains:

```
ClangUserExpression        — high-level expression management
ClangExpressionParser      — wraps clang::CompilerInstance
ClangExpressionDeclMap     — maps LLDB symbols to Clang decls
IRExecutionUnit            — manages JIT-compiled code
ClangFunctionCaller        — calls functions in the target
ClangPersistentVariables   — manages $0, $1, etc.
```

**ClangExpressionParser** creates a `clang::CompilerInstance` configured with:
- The target's triple (so correct ABI rules apply)
- The current frame's local variable types (from DWARF)
- The current module's global symbols
- C++11/14/17 enabled (so modern expressions work)

```cpp
// Simplified pipeline:
ClangUserExpression expr(exe_ctx, expression_str, ...);
expr.Parse(diagnostics_manager, exe_ctx, ...);
// → calls ClangExpressionParser::Parse()
//   → creates clang::CompilerInstance
//   → compiles expression to LLVM IR
//   → links with IRExecutionUnit

expr.Execute(diagnostics_manager, exe_ctx, options, result);
// → allocates code/data in target process via Process::AllocateMemory()
// → JIT-compiles via ORC
// → calls the function in target via Process::WriteMemory + Thread::RunToAddress
// → reads result back
```

### Persistent Variables

LLDB persistent variables (`$0`, `$1`, etc.) are stored in the target process's heap:

```lldb
(lldb) p int x = 42
(int) $0 = 42
(lldb) p $0 + 1
(int) $1 = 43
(lldb) p x     # x from the target's locals
```

`ClangPersistentVariables` maintains a map from variable name to an `lldb_private::ExpressionVariable`. The variable's value lives in memory allocated in the target process, so it participates in the target's heap (accessible via its address) and can be passed to target functions.

### JIT Execution via ORC

`IRExecutionUnit` wraps an ORC JIT instance ([Chapter 108](../part-16-jit-sanitizers/ch108-the-orc-jit.md)):

```cpp
// IRExecutionUnit.cpp (simplified):
llvm::Error IRExecutionUnit::GetRunnableInfo(Status &error,
                                              lldb::addr_t &func_addr) {
    // Get target process memory:
    Process *process = exe_ctx.GetProcessPtr();
    
    // JIT-compile the module:
    auto jit = llvm::orc::LLJITBuilder()
                   .setTargetMachineBuilder(...)
                   .create();
    jit->addIRModule(std::move(tsm));
    
    // Find the entry point:
    auto sym = jit->lookup("$__lldb_expr");
    func_addr = sym->getAddress();
    
    // Copy JIT-compiled code into target process:
    process->WriteMemory(remote_addr, local_code_buf, code_size, error);
    
    return llvm::Error::success();
}
```

The JIT-compiled code calls into the target process's functions directly (since LLDB has mapped the target's memory), making the expression evaluator extremely powerful — you can call arbitrary functions, mutate program state, and observe results.

---

## 116.5 LLDB's Use of LLVM MC

LLDB reuses LLVM's Machine Code (MC) layer for instruction disassembly rather than implementing its own disassembler. This is a significant architectural win: LLDB automatically supports any architecture that LLVM supports, with no extra work.

### Disassembler Plugin

[`lldb/source/Plugins/Disassembler/LLVMC/DisassemblerLLVMC.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lldb/source/Plugins/Disassembler/LLVMC/DisassemblerLLVMC.cpp) wraps `MCDisassembler`:

```cpp
// For: disassemble -b -- <bytes>
DisassemblerLLVMC::Disassemble(Debugger &debugger, const ArchSpec &arch,
                                 const char *plugin_name, ...) {
    // Create MC infrastructure for target arch:
    std::unique_ptr<MCSubtargetInfo> subtarget_info(
        target->createMCSubtargetInfo(triple, cpu, features));
    std::unique_ptr<MCDisassembler> disasm(
        target->createMCDisassembler(*subtarget_info, ctx));
    std::unique_ptr<MCInstPrinter> printer(
        target->createMCInstPrinter(triple, syntax, *asm_info,
                                     *instr_info, *reg_info));
    
    // Disassemble bytes:
    MCInst inst;
    uint64_t size;
    disasm->getInstruction(inst, size, bytes, pc, nulls());
    printer->printInst(&inst, pc, "", *subtarget_info, os);
}
```

### Intel and AT&T Syntax

LLDB supports both AT&T and Intel syntax via the `MCInstPrinter` variant:

```lldb
(lldb) settings set target.x86-disassembly-flavor intel
(lldb) disassemble -n main
main:
    0x401000 <+0>:  push rbp
    0x401001 <+1>:  mov  rbp, rsp

(lldb) settings set target.x86-disassembly-flavor att
(lldb) disassemble -n main
main:
    0x401000 <+0>:  pushq %rbp
    0x401001 <+1>:  movq  %rsp, %rbp
```

### Mixed Source/Disassembly

```lldb
(lldb) disassemble -n foo --mixed
foo:
-> 0x401020 <+0>:   push   rbp
   ; src/foo.cpp:10: int foo(int x) {
   0x401021 <+1>:   mov    rbp, rsp
   ; src/foo.cpp:11:   return x + 1;
   0x401024 <+4>:   lea    eax, [rdi + 1]
```

LLDB correlates instruction addresses to source lines via the DWARF line table, then interleaves source lines with MC-disassembled instructions.

---

## 116.6 Symbol Resolution and DWARF

### SymbolFileDWARF

[`lldb/source/Plugins/SymbolFile/DWARF/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lldb/source/Plugins/SymbolFile/DWARF/) is LLDB's primary debug info plugin. It wraps LLVM's `DWARFContext` and `DWARFUnit` machinery (Chapter 117):

```cpp
// SymbolFileDWARF.cpp:
uint32_t SymbolFileDWARF::ResolveSymbolContext(
    const Address &so_addr, SymbolContextItem resolve_scope,
    SymbolContext &sc) {

    // Find the compilation unit containing this address:
    DWARFUnit *dwarf_cu = GetDWARFCompileUnit(so_addr.GetFileAddress());
    
    // Find the function (DW_TAG_subprogram):
    DWARFDIE func_die = dwarf_cu->LookupAddress(so_addr.GetFileAddress());
    
    // Reconstruct LLDB's Function object from DWARF DIEs:
    sc.function = ParseFunction(*sc.comp_unit, func_die);
    
    // Look up line info:
    sc.line_entry = GetLineEntry(so_addr.GetFileAddress());
    
    return resolved;
}
```

### DIE Caching and Lazy Loading

DWARF files can be gigabytes for large programs. LLDB parses DWARF lazily:

1. On startup, it reads only the `.debug_aranges` (address ranges) section to build a fast address-to-CU index.
2. Individual CUs are parsed only when needed (e.g., when setting a breakpoint by function name or resolving a stack frame).
3. Type information (full struct layouts) is parsed on demand when the expression evaluator needs them.

This lazy loading strategy is critical for debugger startup performance on large binaries.

### TypeSystemClang

For expression evaluation, LLDB reconstructs C++ type information from DWARF and represents it in `clang::ASTContext`. The `TypeSystemClang` plugin manages this:

```cpp
// When the expression evaluator needs the type of 'std::vector<int>':
// 1. SymbolFileDWARF::FindType() searches .debug_info for DW_TAG_structure_type
// 2. ParseStructureDIE() creates clang::RecordDecl with the right fields/methods
// 3. The Clang ASTContext is populated with these reconstructed types
// 4. The expression parser can use them in type checking and code generation
```

### Variable Location Evaluation

DWARF location expressions (Chapter 117) describe where a variable lives at runtime. LLDB evaluates these expressions to determine the variable's address or register:

```cpp
// DWARFExpression.cpp evaluates DW_AT_location:
// Example: DW_OP_fbreg -24 (variable is at frame_base - 24 bytes)
Value result;
DWARFExpression::Evaluate(
    &exe_ctx,       // execution context (has register values, memory access)
    nullptr,        // address resolver
    module,
    opcodes,        // the DW_OP_* byte sequence
    reg_ctx,        // register context for DW_OP_reg*, DW_OP_breg*
    result,
    error);
// result.GetScalar() gives the variable's address or value
```

---

## 116.7 Breakpoints and Watchpoints

### Breakpoint Resolution

LLDB breakpoints are resolved in two stages:

1. **BreakpointSpec** → **BreakpointResolver**: the user's intent (e.g., `break foo` or `break src/foo.cpp:42`)
2. **BreakpointResolver** → **BreakpointLocation list**: actual code addresses where the breakpoint should be placed

```cpp
// Setting a function breakpoint:
lldb::SBBreakpoint bp = target.BreakpointCreateByName("foo", nullptr);
// Internally: BreakpointResolverName searches all symbols for "foo"
// Creates a BreakpointLocation at each matching address

// File/line breakpoint:
lldb::SBBreakpoint bp = target.BreakpointCreateByLocation("src/foo.cpp", 42);
// Internally: BreakpointResolverFileLine searches DWARF line tables
```

When a shared library is loaded dynamically, `DynamicLoader` notifies the `Target`, which re-runs all pending resolvers against the new module. This enables "pending breakpoints" that resolve when the target library is loaded.

### Software Breakpoints

For x86-64, LLDB patches the breakpoint address with a single-byte `0xCC` (`INT3`) instruction:

```cpp
// Process::EnableSoftwareBreakpoint:
uint8_t original_byte;
process.ReadMemory(bp_addr, &original_byte, 1, error);
process.WriteMemory(bp_addr, "\xCC", 1, error);  // INT3
// Save original_byte for restoration when breakpoint is removed
```

On AArch64, LLDB uses `BRK #0` (a 4-byte instruction). When the hardware executes the breakpoint instruction, it generates a SIGTRAP (Linux) or a Mach exception (macOS) that LLDB's `Process` plugin receives and translates into a stop event.

### Hardware Breakpoints and Watchpoints

```lldb
(lldb) breakpoint set -a 0x401000 -H   # hardware breakpoint
(lldb) watchpoint set variable my_var  # hardware watchpoint
(lldb) watchpoint set expression -w write -- &my_var  # write watchpoint
```

Hardware breakpoints use the debug registers (DR0–DR3 on x86, equivalent hardware debug facilities on ARM). LLDB queries the architecture's maximum hardware breakpoint count via the `ABI` plugin and falls back to software breakpoints when hardware resources are exhausted.

Watchpoints use the same hardware debug registers in watchpoint mode. The x86 architecture supports 4 hardware watchpoints of 1, 2, 4, or 8 bytes. LLDB maps user watchpoint requests to these constraints.

### Conditional Breakpoints

```lldb
(lldb) breakpoint set -n foo -c 'x > 100'
```

Conditional breakpoints evaluate a condition expression using the full expression evaluator every time the breakpoint is hit. This is powerful but expensive: each hit pauses the target, evaluates the condition (which may involve a full Clang compilation + JIT if not cached), and resumes if the condition is false. For frequently-hit breakpoints, this can severely slow the target.

LLDB caches the compiled condition expression across hits to the same breakpoint, mitigating the recompilation overhead after the first hit.

---

## 116.8 Remote Debugging and lldb-server

### GDB Remote Serial Protocol

LLDB communicates with `lldb-server` (and any GDB-compatible stub) using the GDB Remote Serial Protocol (RSP). RSP is an ASCII packet protocol over TCP or serial:

```
Client → Server: $qSupported:multiprocess+;xmlRegisters=i386;...#aa
Server → Client: $PacketSize=1000;qXfer:features:read+;...#bb
Client → Server: $?#3f          (why did the target stop?)
Server → Client: $S05#b8        (stopped with signal 5 = SIGTRAP)
Client → Server: $g#67          (read all registers)
Server → Client: $xxxxxxxx...#cc  (register bytes)
Client → Server: $m401000,10#bd  (read 16 bytes at 0x401000)
Server → Client: $48656c6c6f...#cc  (memory bytes)
```

Each packet is framed as `$data#checksum` where checksum is the two-hex-digit sum of all bytes in `data`.

### lldb-server Modes

```bash
# GDB server mode (debugs a specific process):
lldb-server gdbserver 0.0.0.0:1234 -- /path/to/binary arg1 arg2
# Or attach to existing process:
lldb-server gdbserver 0.0.0.0:1234 --attach 1234

# Platform mode (manages remote file transfer, process launch):
lldb-server platform --listen 0.0.0.0:1234 --server

# Connect from lldb client:
(lldb) platform select remote-linux
(lldb) platform connect connect://remote-host:1234
(lldb) target create ./binary
(lldb) run
```

### LLDB Protocol Extensions

LLDB extends RSP with non-standard packets (indicated by lowercase 'q' queries):

| Packet | Description |
|--------|-------------|
| `qProcessInfo` | Process PID, arch, OS |
| `qFileLoadAddress:path` | Address at which file is loaded |
| `jThreadsInfo` | JSON thread info (faster than iterating) |
| `qMemoryRegionInfo:addr` | Memory region permissions |
| `qSaveCore` | Generate core file on server |
| `qGetWorkingDir` | Working directory of target |

These extensions allow LLDB to debug more efficiently than the base RSP protocol permits.

### ProcessGDBRemote

[`lldb/source/Plugins/Process/gdb-remote/ProcessGDBRemote.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lldb/source/Plugins/Process/gdb-remote/ProcessGDBRemote.cpp) implements the `Process` plugin using RSP. Key operations:

```cpp
Status ProcessGDBRemote::DoLaunch(Module *exe_module, ProcessLaunchInfo &launch_info) {
    // Send: $vRun;path;arg1;arg2...
    // Receive: $S05 (stopped at entry)
}

Status ProcessGDBRemote::DoReadMemory(addr_t addr, void *buf, size_t size, ...) {
    // Send: $m<addr>,<size>
    // Receive: $<hex bytes>
}

size_t ProcessGDBRemote::DoWriteMemory(addr_t addr, const void *buf, size_t size, ...) {
    // Send: $M<addr>,<size>:<hex bytes>
}

Status ProcessGDBRemote::DoGetThreadStopReason(lldb::tid_t tid, ...) {
    // Send: $qThreadStopInfo<tid>
    // Receive: JSON stop info
}
```

### QEMU Integration

LLDB connects to QEMU's built-in GDB server for embedded/cross-platform debugging:

```bash
# Start QEMU with GDB server:
qemu-system-aarch64 -M raspi3b -kernel kernel.img \
  -gdb tcp::1234 -S  # -S: start paused

# Connect LLDB:
(lldb) gdb-remote localhost:1234
(lldb) target create --arch aarch64-linux-gnueabi kernel.elf
(lldb) continue
```

LLDB correctly handles QEMU's limited RSP implementation (QEMU doesn't support all LLDB extension packets; `ProcessGDBRemote` falls back gracefully).

---

## 116.9 Python Scripting Interface

### The SB API

LLDB's `lldb` Python module exposes the complete SB (Script Bridge) API. All `lldb::SB*` classes are available:

```python
import lldb

def run_to_crash():
    debugger = lldb.SBDebugger.Create()
    debugger.SetAsync(False)
    
    target = debugger.CreateTarget("/path/to/binary")
    if not target.IsValid():
        print("Failed to create target")
        return
    
    # Set a breakpoint:
    bp = target.BreakpointCreateByName("main")
    
    # Launch:
    process = target.LaunchSimple(None, None, os.getcwd())
    
    # Wait for stop:
    while process.GetState() == lldb.eStateRunning:
        pass
    
    # Examine stopped state:
    thread = process.GetSelectedThread()
    frame = thread.GetSelectedFrame()
    print(f"Stopped in {frame.GetFunctionName()}")
    
    # Read a variable:
    var = frame.FindVariable("argc")
    print(f"argc = {var.GetValue()}")
    
    # Continue:
    process.Continue()
```

### Scripted Breakpoint Commands

```python
# Execute Python when breakpoint is hit:
def my_breakpoint_callback(frame, bp_loc, extra_args, internal_dict):
    # frame: lldb.SBFrame
    # bp_loc: lldb.SBBreakpointLocation
    thread = frame.GetThread()
    process = thread.GetProcess()
    
    # Log a variable:
    var = frame.FindVariable("request_size")
    print(f"request_size = {var.GetValueAsUnsigned()}")
    
    # Don't stop (return True to stop, False to continue):
    return False  # non-stop: just log

# Register in LLDB:
# (lldb) breakpoint command add -F my_module.my_breakpoint_callback 1
```

### Data Formatters

LLDB can display complex types (STL containers, etc.) in a human-readable format via formatters:

```python
# Custom summary for MyType:
def MyType_summary(valobj, internal_dict, options):
    # valobj: lldb.SBValue representing the MyType object
    field_x = valobj.GetChildMemberWithName("x_")
    field_y = valobj.GetChildMemberWithName("y_")
    return f"({field_x.GetValueAsUnsigned()}, {field_y.GetValueAsUnsigned()})"

def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand(
        'type summary add MyType -F myformatters.MyType_summary')
```

LLDB ships with built-in formatters for `std::string`, `std::vector`, `std::map`, `std::unordered_map`, and many other standard library types on both libc++ and libstdc++.

### Scripted Process

LLDB 14+ supports fully scripted process backends, enabling:
- Debugging traces (post-mortem from execution logs)
- Debugging core dumps with custom backends
- Emulated process debugging without a real target

```python
class TracedProcess(lldb.ScriptedProcess):
    def get_memory_region_containing_address(self, addr):
        # Return memory region info from trace
        pass
    
    def read_memory_at_address(self, addr, size, error):
        # Return bytes from trace memory snapshot
        pass
```

---

## Chapter Summary

- **LLDB's design** is library-first (`liblldb`) with a plugin architecture. Every optional capability — binary format parsing, debug info, disassembly, platform integration, process control — is a plugin.

- **The core object hierarchy** is `Debugger → Target → Process → Thread → StackFrame`, with `Module`/`SymbolFile` hanging off `Target`. This hierarchy maps cleanly to the SB API.

- **Expression evaluation** embeds a full Clang compiler that parses and compiles expressions in the context of the current debug frame, generating LLVM IR that is JIT-compiled via ORC and executed in the target process.

- **LLDB reuses LLVM MC** (`MCDisassembler`, `MCInstPrinter`) for all instruction disassembly, gaining automatic support for every LLVM-supported architecture without reimplementation.

- **SymbolFileDWARF** wraps `llvm::DWARFContext` and parses DWARF lazily (CU-by-CU, type-by-type) to minimize startup overhead for large binaries.

- **Breakpoints** use software patching (`INT3`/`BRK #0`) or hardware debug registers. Conditional breakpoints use the expression evaluator (expensive per-hit); compiled conditions are cached after first use.

- **Remote debugging** uses the GDB Remote Serial Protocol with LLDB extensions. `lldb-server` handles both GDB-server mode and platform mode; `ProcessGDBRemote` implements the client side.

- **Python scripting** exposes the complete SB API via the `lldb` module, enabling breakpoint callbacks, custom data formatters, automated debugging scripts, and fully scripted process backends.
