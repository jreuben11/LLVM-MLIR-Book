# Chapter 194 — Zig: Comptime Metaprogramming and LLVM IR Generation

*Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice*

Zig occupies a distinctive position among systems languages: it is simultaneously a replacement for C, a build system for C/C++ projects, and a compiler toolchain distribution. Its central innovation is `comptime` — a first-class metaprogramming primitive that performs arbitrary computation at compile time using the full Zig language, not a restricted subset. Where C++ achieves genericity through template instantiation (a separate, Turing-complete but syntactically distinct sublanguage), and Rust through const generics and `const fn` (powerful but constrained by the borrow checker at compile time), Zig uses the same language, the same types, the same control flow, and the same error propagation for both compile-time and runtime code. The compiler's intermediate representation pipeline — ZIR → AIR → LLVM IR — makes this possible: comptime evaluation occurs during the AIR stage after syntactic ZIR has been produced, so all comptime computation is complete before any LLVM IR is generated. This chapter covers that pipeline in depth, explains how comptime generics, error unions, and safety checks map to LLVM IR, and shows how Zig's bundled Clang/LLD toolchain enables hermetic cross-compilation from any host. Readers should be familiar with LLVM IR from [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) and [Chapter 17 — The Type System](../part-04-llvm-ir/ch17-type-system.md), calling conventions from [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md), and LLD from [Chapter 78 — The LLVM Linker (LLD)](../part-13-lto-whole-program/ch78-lld.md).

---

## 194.1 Architecture: Three Compilation Stages

### The Zig Compiler Pipeline

Zig's self-hosted compiler (written entirely in Zig since version 0.10, replacing the original C++ stage1 bootstrap compiler) processes source files through three intermediate representations before producing native code:

```
Source (.zig)
    │
    ▼
  Parse (src/Parse.zig)
    │  AST — concrete syntax tree with source locations
    ▼
  AstGen (src/AstGen.zig)
    │  ZIR — Zig IR: syntactic, unanalyzed, untyped
    ▼
  Sema (src/Sema.zig)
    │  AIR — Analyzed IR: typed, monomorphized, comptime-evaluated
    ▼
  CodeGen (src/codegen/llvm.zig  OR  src/codegen/c.zig
           OR  src/arch/x86_64/CodeGen.zig, etc.)
    │
    ▼
  LLVM IR → Machine Code   (or: C output, or direct x86/ARM/Wasm assembly)
```

Three front-end compilation modes exist:

- `zig build-exe` — produces a self-contained executable
- `zig build-lib` — produces a static or shared library (`--dynamic` flag)
- `zig build-obj` — produces a single object file; the fundamental unit for incremental compilation

The fourth mode, `zig build`, invokes a `build.zig` program (discussed in §194.7) that orchestrates arbitrary combinations of the above.

### The Self-Hosted Compiler

The Zig bootstrap sequence: a prebuilt `zig1.wasm` binary (a WebAssembly snapshot of the previous compiler release) compiles the Zig self-hosted compiler source into a stage2 binary, which then compiles the full Zig standard library and any user code. This means the only external dependency for building Zig from source is a WebAssembly runtime — by default, Zig ships its own minimal WASM interpreter written in C (`lib/compiler_rt/wasm_helpers.c`). The self-hosted compiler is available at [`src/main.zig`](https://github.com/ziglang/zig/blob/master/src/main.zig) in the Zig source tree.

### Target Discovery

```bash
$ zig version
0.14.0-dev.3+...   # version varies; check ziglang.org/download for stable

$ zig targets | head -30
# Outputs a JSON document listing all supported targets.
# Currently 200+ CPU/OS/ABI combinations.
# Includes: x86_64-linux-gnu, x86_64-linux-musl, aarch64-macos,
#           wasm32-wasi, riscv64-linux-musl, x86_64-windows-gnu, ...
```

Each target triple is a `cpu_arch-os-abi` combination, corresponding to LLVM's target triple system described in [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md).

---

## 194.2 ZIR: The Syntactic IR

### What ZIR Is

ZIR (Zig IR) is produced by `src/AstGen.zig` from the parsed AST. It is:

- **Syntactic** — one ZIR instruction per semantically meaningful AST construct
- **Unanalyzed** — no type inference, no overload resolution, no comptime evaluation
- **Untyped** — types appear as ZIR expressions, not resolved LLVM types
- **Content-addressed** — each ZIR unit is hashed for incremental compilation caching; unchanged functions reuse their cached AIR without re-running Sema

The ZIR instruction set is defined in [`src/Zir.zig`](https://github.com/ziglang/zig/blob/master/src/Zir.zig). Key instructions include:

| ZIR Instruction | Meaning |
|----------------|---------|
| `call` | Function call (unresolved callee) |
| `param` | Function parameter declaration |
| `as_node` | Type coercion (unresolved) |
| `elem_ptr` | Pointer to array/slice element |
| `field_ptr` | Pointer to struct field |
| `array_init` | Array literal |
| `struct_init` | Struct literal |
| `switch_block` | Switch expression with all arms |
| `try` | Error propagation |
| `errdefer_code` | Error-scoped deferred cleanup |
| `compile_error` | Comptime error emission |
| `typeof` | Type of expression (comptime) |
| `type_info` | `@typeInfo` reflection |

### ZIR Usage Outside the Compiler

ZIR is consumed by ZLS (the Zig Language Server) for real-time analysis without needing full semantic analysis. ZLS can process ZIR to provide completions, hover documentation, and error messages without running Sema. This separation is architecturally significant: the Zig toolchain decouples syntactic validation (ZIR) from semantic analysis (AIR), allowing IDE tooling to operate at the fast, cheap layer.

```bash
# Validate ZIR without full compilation:
$ zig ast-check hello.zig

# Dump AIR during compilation (includes ZIR as a side effect):
$ zig build-obj --verbose-air hello.zig

# Format source (uses ZIR for structure):
$ zig fmt hello.zig
```

---

## 194.3 AIR: The Analyzed IR

### From ZIR to AIR

Sema (`src/Sema.zig`) transforms ZIR into AIR by performing:

1. **Type inference**: every value acquires a concrete Zig type
2. **Comptime evaluation**: all `comptime`-qualified expressions are reduced to constants
3. **Monomorphization**: generic functions are instantiated for each unique set of comptime arguments, producing distinct AIR functions
4. **Error set resolution**: error sets are sized and integer-coded
5. **Optional and union layout**: `?T` and `E!T` types get their representation layouts

AIR is a typed SSA IR at the Zig semantic level — it retains Zig's type vocabulary: `i32`, `u8`, `comptime_int`, `?i32`, `!i32`, `[]const u8`, `struct { x: i32, y: i32 }`, and so on. AIR does not yet commit to LLVM's type system. The instruction set is defined in [`src/Air.zig`](https://github.com/ziglang/zig/blob/master/src/Air.zig):

| AIR Instruction | Meaning |
|----------------|---------|
| `arg` | Function argument |
| `alloc` | Stack allocation |
| `store`, `load` | Memory read/write |
| `br`, `cond_br` | Unconditional/conditional branch |
| `block`, `loop` | Structured control flow |
| `call` | Resolved function call |
| `assembly` | Inline assembly |
| `ret`, `ret_ptr` | Return value / return by pointer |
| `add`, `sub`, `mul` | Arithmetic (with safety variants) |
| `div_exact`, `div_trunc`, `div_floor` | Division modes |
| `mod`, `rem` | Modulo/remainder |
| `cmp_eq`, `cmp_lt`, `cmp_gt` | Comparisons |
| `optional_payload` | Unwrap `?T` to `T` |
| `optional_payload_ptr` | Pointer to `?T` payload |
| `err_union_payload` | Unwrap `E!T` to `T` |
| `err_union_code` | Extract error code from `E!T` |
| `unwrap_errunion_err` | Branch on error union variant |

### A Concrete AIR Example

Given this Zig function:

```zig
fn addOne(x: i32) i32 {
    return x + 1;
}
```

The AIR (conceptual dump — actual format is binary with text debug output via `--verbose-air`) looks like:

```
# %0 = arg i32
# %1 = constant i32 1
# %2 = add %0, %1          ; i32
# ret %2
```

All types are fully resolved at this stage. For a generic variant `fn addOne(comptime T: type, x: T) T`, Sema produces separate AIR functions for each `T` that appears at a call site, e.g., `addOne__i32` and `addOne__f64` — full monomorphization, not LLVM-level template expansion.

---

## 194.4 Comptime: The Metaprogramming Primitive

### The `comptime` Qualifier

`comptime` is an annotation that forces compile-time evaluation. It can qualify:

- **Variables**: `comptime var count: usize = 0;` — evaluated and used during compilation
- **Function parameters**: `fn foo(comptime T: type) void { ... }` — type parameter, erased at runtime
- **Expressions**: `comptime blk: { ... }` — arbitrary block evaluated at compile time
- **Fields**: struct fields can have comptime-only types (e.g., `type`)

The comptime evaluator is not a separate interpreter or VM. Sema walks ZIR instructions within a compile-time namespace, maintaining a value environment that maps ZIR slots to compile-time-known constants. When Sema encounters a `comptime`-qualified expression, it evaluates it in this namespace and records the result as a constant AIR value. Errors during comptime evaluation become compile errors at the call site.

### Generic Functions via Comptime Type Parameters

The canonical Zig generics pattern uses `comptime T: type` parameters:

```zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// Each call site produces a distinct monomorphized AIR function:
const x = max(i32, 3, 7);          // monomorphizes max__i32
const y = max(f64, 3.14, 2.71);    // monomorphizes max__f64
```

There is no function template in the LLVM IR output — only `max__i32` and `max__f64` as ordinary concrete functions. Contrast with C++ templates: the template definition exists as a syntactic entity in the AST; Zig's comptime generics exist only as ZIR until instantiation, then produce AIR directly.

### `type` as a First-Class Value

In Zig, `type` is a comptime-only type whose values are types. This enables type-level computation:

```zig
fn VecType(comptime T: type, comptime N: usize) type {
    return [N]T;
}

const Vec3f = VecType(f32, 3);   // comptime: Vec3f == [3]f32
const Vec4i = VecType(i32, 4);   // comptime: Vec4i == [4]i32

var a: Vec3f = .{ 1.0, 2.0, 3.0 };
var b: Vec4i = .{ 0, 1, 2, 3 };
```

`VecType` is called at comptime, returns a type, and that type is used in variable declarations. No runtime code corresponds to `VecType` itself — it is fully erased.

### Reflection with `@typeInfo`

`@typeInfo(T)` returns a `std.builtin.Type` tagged union at comptime. This is the mechanism underlying all Zig generic algorithms:

```zig
const std = @import("std");
const builtin = @import("builtin");

fn printTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);
    switch (info) {
        .Int => |i| {
            // i.signedness: std.builtin.Signedness (.signed or .unsigned)
            // i.bits: u16
            std.debug.print("Int: {s} {d} bits\n", .{
                @tagName(i.signedness), i.bits,
            });
        },
        .Struct => |s| {
            std.debug.print("Struct with {d} fields:\n", .{s.fields.len});
            inline for (s.fields) |field| {
                std.debug.print("  .{s}: {s}\n", .{
                    field.name, @typeName(field.type),
                });
            }
        },
        .Optional => |o| {
            std.debug.print("Optional of {s}\n", .{@typeName(o.child)});
        },
        .ErrorUnion => |eu| {
            std.debug.print("ErrorUnion: {s}!{s}\n", .{
                @typeName(eu.error_set), @typeName(eu.payload),
            });
        },
        else => std.debug.print("Other type: {s}\n", .{@typeName(T)}),
    }
}

comptime {
    // Executed at compile time — errors become compile errors
    printTypeInfo(u32);       // "Int: unsigned 32 bits"
    printTypeInfo(?[]u8);     // "Optional of []u8"
}
```

The full `std.builtin.Type` union has variants: `.Type`, `.Void`, `.Bool`, `.NoReturn`, `.Int`, `.Float`, `.Pointer`, `.Array`, `.Struct`, `.ComptimeFloat`, `.ComptimeInt`, `.Undefined`, `.Null`, `.Optional`, `.ErrorUnion`, `.ErrorSet`, `.Enum`, `.Union`, `.Fn`, `.Opaque`, `.Frame`, `.AnyFrame`, `.Vector`, `.EnumLiteral`.

### Comptime Reflection Intrinsics Reference

| Intrinsic | Return type | Meaning |
|-----------|------------|---------|
| `@typeInfo(T)` | `std.builtin.Type` | Full type descriptor (comptime) |
| `@typeName(T)` | `*const [N:0]u8` | Type name as comptime string |
| `@hasField(T, "name")` | `bool` | Test for struct/union field |
| `@hasDecl(T, "name")` | `bool` | Test for decl in container |
| `@field(val, "name")` | field type | Runtime field access by comptime name |
| `@sizeOf(T)` | `comptime_int` | Byte size of type |
| `@alignOf(T)` | `comptime_int` | ABI alignment of type |
| `@bitSizeOf(T)` | `comptime_int` | Bit size of type |
| `@offsetOf(T, "f")` | `comptime_int` | Byte offset of field in struct |
| `@tagName(val)` | `[:0]const u8` | Name of enum/union tag |

### Inline For and Inline While

`inline for` unrolls a loop at compile time over tuple types or enum fields. It is the mechanism for generating code over heterogeneous type sequences:

```zig
fn printFields(val: anytype) void {
    const T = @TypeOf(val);
    const info = @typeInfo(T).Struct;
    inline for (info.fields) |field| {
        // field.name is a comptime string; @field is a comptime field accessor
        std.debug.print("{s} = {any}\n", .{
            field.name, @field(val, field.name),
        });
    }
}

const Point = struct { x: f32, y: f32, z: f32 };
const p = Point{ .x = 1.0, .y = 2.0, .z = 3.0 };
printFields(p);
// Emits: x = 1.0 / y = 2.0 / z = 3.0
// The loop is fully unrolled; no loop exists in the output AIR
```

`inline while` provides comptime iteration when the termination condition is not known at the start:

```zig
comptime var i: usize = 0;
inline while (i < 4) : (i += 1) {
    // Body executes 4 times at comptime
}
```

### Comparison: Zig Comptime vs. C++ Templates vs. Rust const

| Feature | Zig comptime | C++ templates | Rust `const fn` / generics |
|---------|-------------|---------------|---------------------------|
| Mechanism | Interpreter in Sema | Template substitution | Const evaluator + monomorphization |
| Heap allocation at CT | Yes (via comptime allocator) | No | No (stable Rust) |
| Error messages | Regular Zig errors at call site | Template instantiation errors | Trait bound errors |
| Turing complete | Yes | Yes (via SFINAE/concepts) | Yes (via `const fn` recursion) |
| Separate sublanguage | No | Yes (template syntax) | Partially (`const` annotation) |
| Runtime code emitted | No (fully erased) | No (fully instantiated) | No (monomorphized) |
| Dynamic dispatch | Through `anytype` + `@typeInfo` | Through virtual/CRTP | Through trait objects |

Zig's key advantage over C++ is that comptime evaluation runs the same Zig code the programmer already knows — there is no separate template meta-programming layer with distinct syntax and error semantics. Its advantage over Rust const generics is that comptime can perform arbitrary heap-allocating computation (sorting a comptime array, constructing a comptime hash map of string-to-function-pointer mappings, etc.) whereas Rust's const evaluation is restricted to stack-bounded computation.

### Comptime HashMap Key Validation Example

```zig
const std = @import("std");

// Comptime-validated dispatch table: ensures all keys are known at compile time
fn makeDispatcher(comptime handlers: anytype) type {
    // handlers is a tuple of .{ "name", handlerFn } pairs
    // Validate at comptime that all names are non-empty strings
    const fields = @typeInfo(@TypeOf(handlers)).Struct.fields;
    comptime {
        for (fields) |field| {
            const pair = @field(handlers, field.name);
            const key: []const u8 = pair[0];
            if (key.len == 0) {
                @compileError("Dispatcher key must be non-empty");
            }
        }
    }
    return struct {
        pub fn dispatch(key: []const u8) ?void {
            inline for (fields) |field| {
                const pair = @field(handlers, field.name);
                if (std.mem.eql(u8, key, pair[0])) {
                    pair[1]();
                    return {};
                }
            }
            return null;
        }
    };
}

fn handleFoo() void { std.debug.print("foo\n", .{}); }
fn handleBar() void { std.debug.print("bar\n", .{}); }

const D = makeDispatcher(.{
    .{ "foo", handleFoo },
    .{ "bar", handleBar },
});
// D.dispatch("foo") calls handleFoo(); D.dispatch("baz") returns null
// The inline for unrolls into a sequence of string comparisons — no hash table at runtime
```

### Comptime Format String Validation

Perhaps the most practical demonstration of Zig's comptime power is `std.fmt.format` — the function behind `std.debug.print`. The format string `"{s} is {d} bytes\n"` is validated at compile time against the argument tuple types. This validation is ordinary Zig code running in Sema, not a special compiler primitive:

```zig
// Simplified conceptual sketch of std.fmt.format's comptime validation:
fn validateFormatString(comptime fmt: []const u8, comptime Args: type) void {
    const args_info = @typeInfo(Args).Struct.fields;
    comptime var arg_idx: usize = 0;
    comptime var i: usize = 0;
    inline while (i < fmt.len) : (i += 1) {
        if (fmt[i] == '{') {
            if (i + 1 < fmt.len and fmt[i + 1] == '{') {
                i += 1;  // escaped brace
                continue;
            }
            // Find the closing '}'
            const close = comptime blk: {
                var j = i + 1;
                while (j < fmt.len and fmt[j] != '}') : (j += 1) {}
                break :blk j;
            };
            const spec = fmt[i + 1 .. close];
            if (arg_idx >= args_info.len) {
                @compileError("too few arguments for format string");
            }
            // Validate spec against args_info[arg_idx].type:
            const ArgT = args_info[arg_idx].type;
            switch (spec[0]) {
                'd', 'i' => if (@typeInfo(ArgT) != .Int and @typeInfo(ArgT) != .ComptimeInt)
                    @compileError("expected integer for 'd' specifier"),
                's' => if (@typeInfo(ArgT) != .Pointer)
                    @compileError("expected string slice for 's' specifier"),
                else => {},  // other specifiers handled similarly
            }
            arg_idx += 1;
            i = close;
        }
    }
    if (arg_idx != args_info.len) {
        @compileError("too many arguments for format string");
    }
}
```

The actual `std.fmt` implementation handles far more specifiers (`{any}`, `{x}`, `{b}`, `{e}`, custom formatter interfaces via `pub fn format(...)` on types), but the mechanism is identical: `inline while` over a comptime-known string, `@typeInfo` on argument types, and `@compileError` to turn type mismatches into compile errors. When the programmer writes:

```zig
std.debug.print("value = {d}\n", .{"not a number"});
// compile error: expected integer for 'd' specifier
```

The error occurs at the `std.debug.print` call site, with a message pointing at the format string. There is no runtime crash — the type mismatch is a compile error. This is the defining property of Zig's comptime model: compile-time errors are the same as runtime errors, just triggered during Sema.

---

## 194.5 Error Handling

### Error Sets and Error Union Types

Zig's errors are not exceptions — they are values. An **error set** is a closed enumeration of error names, sized as a `u16` by default:

```zig
const FileError = error{
    NotFound,
    PermissionDenied,
    OutOfMemory,
};
```

An **error union type** `E!T` is a tagged union whose tag is either "ok" (carrying a `T` payload) or an error from set `E` (carrying a `u16` error code). The representation is:

- If `T` has a valid sentinel that cannot occur in the ok case, a sentinel-optimization collapses the union to just `T` (analogous to Rust's `NonZero` niche optimization).
- Otherwise: `struct { code: u16, payload: T }` where `code == 0` means "ok".

```zig
fn openFile(path: []const u8) FileError!std.fs.File {
    return std.fs.cwd().openFile(path, .{}) catch |err| switch (err) {
        error.FileNotFound => error.NotFound,
        error.AccessDenied => error.PermissionDenied,
        else => error.OutOfMemory,
    };
}
```

### `try`, `catch`, and `errdefer`

```zig
fn processFile(path: []const u8) !void {
    const file = try openFile(path);   // propagates error if openFile returns error
    defer file.close();                // runs unconditionally on scope exit

    const data = try file.readToEndAlloc(allocator, 1024 * 1024);
    errdefer allocator.free(data);     // runs ONLY if returning an error below

    try parseAndProcess(data);
    allocator.free(data);              // reached only if parseAndProcess succeeds
}
```

`try expr` desugars to `expr catch |e| return e` — it propagates the error up the call stack. `errdefer` runs only on error returns from the enclosing scope, not on successful returns. This enables a pattern where cleanup is registered immediately after allocation, without the branching that C cleanup patterns require.

### Error Return Traces

Zig's error return traces are a lightweight alternative to full stack traces. When `zig build` uses `Debug` or `ReleaseSafe` mode:

- Each `return error.Foo` in a function records the return address into a thread-local ring buffer
- The ring buffer is a `std.builtin.StackTrace` struct (an array of return addresses plus an index)
- When `main` receives an error, `std.debug.dumpCurrentStackTrace` walks the ring buffer and resolves addresses to source locations using debug info

This is cheaper than a full stack trace because:
1. Only error-returning call sites are recorded, not every call frame
2. The ring buffer is fixed-size and pre-allocated
3. Address resolution happens at crash time, not at each error propagation point

```zig
pub fn main() !void {
    processFile("missing.txt") catch |err| {
        std.debug.print("Error: {}\n", .{err});
        // Prints error return trace showing each `try` that propagated the error
        return err;
    };
}
```

### `anyerror` — The Global Error Set

`anyerror` is the union of all error sets in the entire program (computed at compile time). A function returning `anyerror!T` can return any error value ever defined in the build. This is useful for function pointers and interface types:

```zig
const Handler = *const fn ([]const u8) anyerror!void;
```

---

## 194.6 LLVM IR Generation

### Type Mapping: Zig Types to LLVM Types

The lowering from AIR to LLVM IR is performed in [`src/codegen/llvm.zig`](https://github.com/ziglang/zig/blob/master/src/codegen/llvm.zig) using the LLVM C API (`llvm-c/Core.h`). The core mapping function `toLlvmType` handles:

| Zig type | LLVM type | Notes |
|----------|-----------|-------|
| `i8`, `u8`, `i16`, `u16`, `i32`, `u32`, `i64`, `u64` | `i8`, `i8`, `i16`, `i16`, `i32`, `i32`, `i64`, `i64` | Signedness carried in AIR, not LLVM |
| `f16`, `f32`, `f64`, `f128` | `half`, `float`, `double`, `fp128` | |
| `bool` | `i1` | |
| `void` | `void` | |
| `?T` (optional, no sentinel) | `{ T, i1 }` | `i1` = is_non_null |
| `?*T` (optional pointer) | `ptr` | Null pointer represents null |
| `E!T` (error union) | `{ i16, T }` | `i16` = error code (0 = ok) |
| `*T`, `[*]T`, `[]T` | `ptr` | Opaque pointers (LLVM 15+ `ptr`) |
| `struct { ... }` | `{ field_types... }` | Packed struct: `<{ ... }>` |
| `enum(T) { ... }` | `T` (the tag integer) | |
| `union(enum) { ... }` | `{ tag_type, [N x i8] }` | N = max field size, aligned |
| `[N]T` | `[N x T]` | |
| `fn(...) T` | `T(...)` → `ptr` (function pointer) | |

**Pointer semantics**: Since Zig targets LLVM 15+, all pointer types — `*T`, `[*]T`, `[]T.ptr`, `*const T`, etc. — compile to LLVM's opaque `ptr` type. Pointer element types are tracked in AIR but erased during lowering, matching LLVM's opaque pointer model discussed in [Chapter 17 — The Type System](../part-04-llvm-ir/ch17-type-system.md).

### LLVM IR for Optional Return Type

Consider this Zig function:

```zig
fn divide(a: i32, b: i32) ?i32 {
    if (b == 0) return null;
    return @divTrunc(a, b);
}
```

The AIR represents the return type as `?i32` — an optional integer. The `toLlvmType(?i32)` call produces `{ i32, i1 }`. The emitted LLVM IR (from `zig build-exe -femit-llvm-ir=output.ll` followed by inspection of `output.ll`) looks approximately like:

```llvm
; Zig optional: ?i32 → { i32, i1 }
; Returned via sret pointer in some ABIs, or directly as { i32, i1 } in others

define { i32, i1 } @divide(i32 %a, i32 %b) {
entry:
  %is_zero = icmp eq i32 %b, 0
  br i1 %is_zero, label %return_null, label %do_div

return_null:
  ; Return null: payload = undef, is_non_null = false
  %null_val = insertvalue { i32, i1 } undef, i1 false, 1
  ret { i32, i1 } %null_val

do_div:
  ; @divTrunc compiles to sdiv (truncation toward zero)
  %result = sdiv i32 %a, %b
  ; Return Some(result): payload = result, is_non_null = true
  %some_0 = insertvalue { i32, i1 } undef, i32 %result, 0
  %some_1 = insertvalue { i32, i1 } %some_0, i1 true, 1
  ret { i32, i1 } %some_1
}
```

Note the use of `insertvalue` to construct the `{ i32, i1 }` aggregate. In practice, Zig's LLVM codegen may use `alloca` + `store` for larger aggregates or when the ABI requires pointer-passing; the exact form depends on the target calling convention (see [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md)).

### Error Union IR

For `FileError!i32`:

```llvm
; { i16, i32 } — error code (0 = ok) + payload
; Returned as aggregate or via sret depending on size and ABI

define { i16, i32 } @readCount() {
entry:
  ; Success path: error code 0, payload = 42
  %ok_0 = insertvalue { i16, i32 } undef, i16 0, 0
  %ok_1 = insertvalue { i16, i32 } %ok_0, i32 42, 1
  ret { i16, i32 } %ok_1

error_path:
  ; Error path: error code for OutOfMemory (e.g., 2), payload = undef
  %err_0 = insertvalue { i16, i32 } undef, i16 2, 0
  ret { i16, i32 } %err_0
}
```

### Tagged Union IR

Zig's `union(enum) { ... }` compiles to a struct containing the tag (smallest integer type that can represent all variants) plus a payload buffer sized and aligned to the largest field:

```zig
const Value = union(enum) {
    int: i32,
    float: f64,
    boolean: bool,
};
// LLVM type: { i2, [8 x i8] }
// tag: i2 (3 variants), payload: 8 bytes (size of f64), align 8
```

The payload is accessed by bitcasting the `[8 x i8]` buffer to the appropriate field type, which in opaque-pointer LLVM IR uses `load` with a type annotation.

### Packed Structs and Bit-Level Layout

Zig's `packed struct` maps to LLVM's packed struct type `<{ ... }>` with `align(1)`. Fields with non-byte-aligned sizes are bit-packed into the smallest enclosing integer. This is where Zig diverges sharply from C, which does not portably support non-byte-aligned fields:

```zig
const Flags = packed struct {
    active: bool,       // 1 bit
    priority: u3,       // 3 bits
    reserved: u4,       // 4 bits
    // Total: 8 bits = 1 byte
};

const Header = packed struct {
    magic: u16,
    flags: Flags,      // 1 byte, packed immediately after magic
    length: u32,
    // Total: 7 bytes (no padding)
};
```

The LLVM type for `Header` is `<{ i16, i8, i32 }>` — a packed struct that prevents the compiler from inserting padding between fields. The LLVM IR for accessing `header.flags.priority` uses a shift-and-mask sequence:

```llvm
; Load the flags byte:
%flags_byte = load i8, ptr %flags_ptr, align 1
; Extract the priority field (bits 1-3):
%shifted = lshr i8 %flags_byte, 1
%priority = and i8 %shifted, 7     ; mask to 3 bits
```

The `toLlvmType` function in `src/codegen/llvm.zig` special-cases `packed struct` to emit LLVM packed struct syntax and sets the `align(1)` attribute on all loads and stores from packed fields. Non-packed structs use the natural LLVM alignment rules, which the ABI mandates.

### `@Vector` — SIMD Vector Types

Zig's `@Vector(N, T)` type maps directly to LLVM's `<N x T>` vector type, enabling SIMD operations through ordinary Zig arithmetic operators when the vector type is used:

```zig
fn addVectors(a: @Vector(4, f32), b: @Vector(4, f32)) @Vector(4, f32) {
    return a + b;   // comptime: knows both are @Vector(4, f32)
}
```

This lowers to LLVM IR that uses vector arithmetic:

```llvm
define <4 x float> @addVectors(<4 x float> %a, <4 x float> %b) {
entry:
  %result = fadd <4 x float> %a, %b
  ret <4 x float> %result
}
```

The LLVM backend selects the appropriate SIMD instruction for the target: `vaddps ymm0, ymm1, ymm2` on x86-64 with AVX, `fadd v0.4s, v1.4s, v2.4s` on AArch64 NEON. Zig exposes `@splat(N, scalar)` to broadcast a scalar into a vector, and element access uses standard array indexing syntax:

```zig
const v: @Vector(4, i32) = .{ 1, 2, 3, 4 };
const doubled = v * @as(@Vector(4, i32), @splat(2));
const first: i32 = doubled[0];  // 2
```

No intrinsic function names or platform-specific headers are required — the type system carries the vector width and element type through the entire compilation pipeline from AIR to LLVM IR to machine code selection.

### Emitting LLVM IR

```bash
# Emit LLVM IR for inspection:
$ zig build-exe -femit-llvm-ir=output.ll hello.zig

# Emit bitcode:
$ zig build-exe -femit-llvm-bc=output.bc hello.zig

# Emit assembly:
$ zig build-exe -femit-asm=output.s hello.zig

# Cross-compile and emit IR for a different target:
$ zig build-obj -target aarch64-linux-musl -femit-llvm-ir=arm64.ll hello.zig
```

---

## 194.7 Safety Builds and Runtime Checks

### The Four Optimization Modes

Zig's build modes determine the tradeoff between safety checks and optimization. They are specified in `build.zig` via `std.builtin.OptimizeMode` or on the command line with `-O`:

| Mode | CLI Flag | Safety Checks | Optimization | Use Case |
|------|----------|--------------|--------------|----------|
| `Debug` | `-O Debug` (default) | Enabled | None (`-O0`) | Development; catches all UB at runtime |
| `ReleaseSafe` | `-O ReleaseSafe` | Enabled | `-O2` | Production code requiring safety guarantees |
| `ReleaseFast` | `-O ReleaseFast` | Disabled | `-O3 -ffast-math` | Maximum throughput; UB is programmer's responsibility |
| `ReleaseSmall` | `-O ReleaseSmall` | Disabled | `-Oz` | Embedded/size-constrained targets |

### Runtime Safety Checks

In `Debug` and `ReleaseSafe`, Zig inserts checks that would be undefined behavior in C:

**Integer overflow**: Every arithmetic operation uses Zig's checked arithmetic intrinsics before emitting the raw LLVM arithmetic instruction. For example, `a + b` on `i32` in safe mode compiles to:

```llvm
; Zig: const result = a + b;  (in Debug/ReleaseSafe)
%overflow = call { i32, i1 } @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
%val = extractvalue { i32, i1 } %overflow, 0
%did_overflow = extractvalue { i32, i1 } %overflow, 1
br i1 %did_overflow, label %panic, label %ok

panic:
  ; call Zig's panic handler with "integer overflow" message
  call void @panic_integer_overflow(...)
  unreachable

ok:
  ; use %val
```

In `ReleaseFast`, the same `a + b` emits `add i32 %a, %b` with `nsw` (no signed wrap) set, allowing LLVM to optimize assuming no overflow — undefined behavior if it occurs.

**Array/slice out-of-bounds**: Before every `array[index]` or `slice[index]`, safe builds emit:

```llvm
%in_bounds = icmp ult i64 %index, %len
br i1 %in_bounds, label %ok, label %panic_oob
```

**Null pointer dereference**: Before dereferencing `?*T`, safe builds check `is_non_null`.

**`unreachable` code**: In `Debug`/`ReleaseSafe`, `unreachable` compiles to a `@panic("reached unreachable code")` call. In `ReleaseFast`, it compiles to `llvm.trap` or simply `unreachable` (the LLVM IR instruction), which is UB if executed.

**Union tag mismatch**: Accessing the wrong field of a `union(enum)` panics in safe builds:

```llvm
%tag = load i2, ptr %union_tag_ptr
%expected_tag = i2 0          ; e.g., accessing .int field
%tag_ok = icmp eq i2 %tag, %expected_tag
br i1 %tag_ok, label %access, label %panic_wrong_tag
```

**Division by zero and exact division failure**: `@divTrunc(a, b)` in safe mode checks `b != 0`; `@divExact(a, b)` additionally checks that `a % b == 0`.

### Per-Block Safety Opt-Out

For performance-critical inner loops, Zig allows disabling safety checks at block granularity:

```zig
fn fastSum(data: []const i32) i64 {
    var sum: i64 = 0;
    @setRuntimeSafety(false);  // disable for this block
    for (data) |x| {
        sum += x;   // no overflow check — programmer guarantees no overflow
    }
    return sum;
}
```

This is more granular than Rust's `unsafe` blocks (which apply to memory safety, not arithmetic overflow checks) and more granular than C's absence of any overflow checking at all.

### Comparison with Rust Debug/Release

Rust also performs integer overflow checks in debug mode and wraps (or panics, depending on the `overflow-checks` profile setting) in release. Zig's model is more explicit: the four named modes are self-describing in `build.zig`, and the per-block `@setRuntimeSafety` opt-out gives finer control than Rust's profile-level `overflow-checks = false`. Zig also checks division by zero by default in safe builds; Rust requires explicit use of `checked_div`.

---

## 194.8 Zig as a C Cross-Compiler

### The Bundled Toolchain

Zig ships a complete, hermetic toolchain that includes:

- **Clang/LLVM**: for compiling C and C++ source files
- **LLD**: the LLVM linker (see [Chapter 78 — The LLVM Linker (LLD)](../part-13-lto-whole-program/ch78-lld.md))
- **musl libc** source: compiled from source for Linux musl targets
- **glibc stubs**: precompiled for specific glibc versions (to avoid sysroot dependency)
- **libunwind + libcxx**: for C++ exception handling and standard library
- **macOS SDK wrappers** and **Windows SDK wrappers**: for cross-compilation to Darwin and Win32

This means `zig cc` requires no host sysroot, no host libc headers, and no system linker. The target triple specifies everything:

```bash
# Cross-compile a C file for static Linux (musl), from macOS or Windows host:
$ zig cc -target x86_64-linux-musl -O2 -o hello hello.c

# Cross-compile for ARM64 Linux with glibc 2.31:
$ zig cc -target aarch64-linux-gnu.2.31 -O2 -o hello hello.c

# Cross-compile for Windows with MinGW ABI:
$ zig cc -target x86_64-windows-gnu -O2 -o hello.exe hello.c

# Drop-in replacement for gcc in Makefile:
$ CC="zig cc" make
```

### Production Usage

The hermetic cross-compilation capability has been adopted in production systems:

- **TigerBeetle** ([tigerbeetle.com](https://tigerbeetle.com)): financial database written in Zig; uses `zig build` for all targets
- **Bun** ([bun.sh](https://bun.sh)): JavaScript runtime; uses `zig cc` to cross-compile JavaScriptCore (a large C++ codebase) for Linux/macOS/Windows from a single CI host, eliminating per-platform build machines
- **ffmpeg** (community build scripts): uses `zig cc` for reproducible static binary cross-compilation

The Andrew Kelley LLVM Dev Meeting 2022 talk ([slides](https://llvm.org/devmtg/2022-11/slides/)) covers the Zig build system integration with LLVM in detail.

---

## 194.9 C Interoperability

### `@cImport` — C Header Translation

Zig imports C headers at comptime via Clang's AST. The `@cImport` block is evaluated during Sema; Clang parses the specified headers and translates the resulting AST to Zig declarations.

Consider a minimal C library header:

```c
/* mylib.h */
#ifndef MYLIB_H
#define MYLIB_H

#include <stddef.h>
#include <stdint.h>

typedef struct {
    uint32_t id;
    float    value;
    char     label[32];
} MyRecord;

/* Returns 0 on success, negative errno on failure. */
int mylib_process(const MyRecord *records, size_t count, uint32_t flags);

/* Initializes the library; must be called before mylib_process. */
void mylib_init(void);

#endif /* MYLIB_H */
```

The Zig code that imports and calls this C header:

```zig
const c = @cImport({
    @cDefine("MYLIB_EXTRA", "1");
    @cInclude("mylib.h");
    // Can include multiple headers; all accumulate into the `c` namespace
    @cInclude("stdio.h");
});

pub fn main() !void {
    c.mylib_init();

    const records = [_]c.MyRecord{
        .{ .id = 1, .value = 3.14, .label = [_]u8{0} ** 32 },
        .{ .id = 2, .value = 2.71, .label = [_]u8{0} ** 32 },
    };
    const ret = c.mylib_process(&records[0], records.len, 0);
    if (ret != 0) {
        _ = c.printf("mylib_process failed: %d\n", ret);
        return error.ProcessFailed;
    }
}
```

`@cDefine("MACRO", "value")` sets a preprocessor macro before the includes; `@cUndef("MACRO")` undefines one. The resulting `c` namespace contains all C declarations as Zig types: C structs become `extern struct`, C enums become `extern enum`, function declarations become `extern fn`, and pointer types use Zig's pointer vocabulary. Notably, `MyRecord` is translated as an `extern struct` with the exact C layout, including the `[32]u8` fixed-size array for `label`.

### `zig translate-c`

For exploring C library APIs, `zig translate-c` emits a Zig file containing all declarations from a C header:

```bash
$ zig translate-c /usr/include/stdio.h > stdio_zig.zig
# Produces Zig declarations for all public symbols in stdio.h
```

This is useful for auditing the translation and for generating Zig bindings for C libraries.

### `extern` Types and ABI Compatibility

```zig
// C-compatible struct (same layout as struct sockaddr_in in C):
const SockaddrIn = extern struct {
    sin_family: c_ushort,
    sin_port: u16,
    sin_addr: extern struct { s_addr: u32 },
    sin_zero: [8]u8,
};

// C-compatible function pointer type:
const Callback = *const fn (c_int, [*:0]const u8) callconv(.C) void;

// Declare an external C function:
extern fn pthread_create(
    thread: *c_ulong,
    attr: ?*const anyopaque,
    start_routine: *const fn (?*anyopaque) callconv(.C) ?*anyopaque,
    arg: ?*anyopaque,
) c_int;
```

The `callconv(.C)` annotation ensures the C calling convention is used, which on x86-64 Linux corresponds to the System V AMD64 ABI. Cross-language ABI patterns are covered in depth in [Chapter 196 — Cross-Language ABI Interoperability](../part-28-language-ecosystems/ch196-cross-language-abi.md).

---

## 194.10 The Build System (`build.zig`)

### `build.zig` as a Zig Program

Unlike CMake (a DSL), Make (a rule-evaluation engine), or Bazel (a sandboxed build language), Zig's build system is a Zig program. `build.zig` imports `std.Build` and constructs a directed acyclic graph of `std.Build.Step` objects. When `zig build` runs, it evaluates the DAG lazily — only steps reachable from the requested target (`zig build run`, `zig build test`, etc.) are executed.

### Minimal `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "hello",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Add a C source file to the same compilation unit:
    exe.addCSourceFile(.{
        .file = b.path("src/helper.c"),
        .flags = &.{"-std=c17", "-O2"},
    });

    // Link against the system's libz:
    exe.linkSystemLibrary("z");
    exe.linkLibC();

    b.installArtifact(exe);

    // Add a run step: `zig build run`
    const run_cmd = b.addRunArtifact(exe);
    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);

    // Add a test step: `zig build test`
    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&b.addRunArtifact(unit_tests).step);
}
```

### Content-Addressed Caching

`std.Build.Cache` implements content-addressed caching: each build artifact is keyed by a hash of its inputs (source files, compiler flags, target, and Zig version). Unchanged artifacts are reused from the cache at `~/.cache/zig/` (Linux/macOS) or `%LOCALAPPDATA%\zig\` (Windows). This caching is what makes incremental rebuilds fast: ZIR content-addressing at the module level means only files with changed ZIR are re-analyzed.

### Key `std.Build` API Surface

| API | Purpose |
|-----|---------|
| `b.addExecutable(...)` | Create an executable compilation step |
| `b.addStaticLibrary(...)` | Create a static library |
| `b.addSharedLibrary(...)` | Create a shared library |
| `b.addObject(...)` | Create a single object file |
| `b.addTest(...)` | Create a test binary |
| `exe.addCSourceFile(...)` | Add a C/C++ source file |
| `exe.addIncludePath(...)` | Add an include path |
| `exe.linkSystemLibrary(...)` | Link against a system library |
| `exe.linkLibC()` | Link against the C runtime |
| `b.dependency(...)` | Fetch a package from the package index |
| `b.step("name", "desc")` | Define a named build target |
| `b.installArtifact(...)` | Install an artifact to the prefix |
| `b.standardTargetOptions(...)` | Parse `-Dtarget=` from CLI |
| `b.standardOptimizeOption(...)` | Parse `-Doptimize=` from CLI |

### Lazy Dependency Evaluation

`b.dependency("pkg", .{})` fetches a package declared in `build.zig.zon` (the package manifest, analogous to `Cargo.toml`). Dependencies are fetched lazily — only if the step that uses them is in the build graph for the current invocation. This means a `zig build test` that only exercises pure-Zig code does not fetch C dependencies used only by a WASM target.

### Built-in Test Framework

Zig's test framework is built into the language and the `zig build test` command; no external test harness is required. Test blocks are declared with the `test "description" { ... }` syntax anywhere in a `.zig` file:

```zig
const std = @import("std");
const expect = std.testing.expect;
const expectEqual = std.testing.expectEqual;
const expectError = std.testing.expectError;

fn fibonacci(n: u32) u32 {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

test "fibonacci base cases" {
    try expectEqual(@as(u32, 0), fibonacci(0));
    try expectEqual(@as(u32, 1), fibonacci(1));
}

test "fibonacci sequence" {
    try expectEqual(@as(u32, 8), fibonacci(6));
    try expectEqual(@as(u32, 55), fibonacci(10));
}

test "error propagation" {
    const result = openFile("nonexistent.txt");
    try expectError(error.NotFound, result);
}
```

`std.testing.expectEqual` and `std.testing.expect` use `try` to propagate test failures as errors. The test binary is compiled in `Debug` mode by default (to catch safety violations), but the optimize mode can be overridden in `build.zig`. Test discovery is static: the compiler identifies all `test` blocks in the files reachable from the test root and includes them in the binary. `zig test file.zig` compiles and runs tests in a single file; `b.addTest(...)` in `build.zig` integrates testing into the build graph so that `zig build test` runs all tests across the project.

---

## 194.11 Integration with LLVM's Optimization Pipeline

### What Zig Passes to LLVM

After AIR-to-LLVM IR lowering, Zig invokes the LLVM optimization pipeline through the C API. The pass pipeline used depends on the optimization mode:

- `Debug`: no optimization passes; LLVM IR is emitted and passed directly to the backend
- `ReleaseSafe`: `PassBuilder` with `O2` pipeline; no fast-math (preserves IEEE semantics)
- `ReleaseFast`: `PassBuilder` with `O3` pipeline; `FastMathFlags` set on all FP operations
- `ReleaseSmall`: `PassBuilder` with `Oz` pipeline

Zig uses the new LLVM pass manager (the `PassBuilder` API introduced in LLVM 12 and stabilized in LLVM 15), not the legacy pass manager. LTO is supported via `--lto=thin` or `--lto=full` flags in `build.zig`.

### Sanitizer Integration

Zig supports enabling LLVM sanitizers through build options:

```zig
// In build.zig:
exe.sanitize_thread = true;   // TSan
exe.sanitize_memory = false;  // MSan (experimental)
```

And via `zig cc`:

```bash
$ zig cc -fsanitize=address -fsanitize=undefined -o prog prog.c
```

ASan, UBSan, and TSan are supported through LLVM's sanitizer pass instrumentation, the same mechanism described in [Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md). On AArch64 targets, Zig can also benefit from Hardware Address Sanitizer (HWASan) and ARM Memory Tagging Extension (MTE) support, where the LLVM backend emits MTE tag instructions rather than shadow-memory loads; this is covered in [Chapter 111 — HWASan and MTE](../part-16-jit-sanitizers/ch111-hwasan-mte.md). Note that Zig's own runtime safety checks (§194.7) catch most of the same bugs as UBSan in Debug mode, making UBSan redundant for pure Zig code; sanitizers are primarily useful when mixing Zig and C code or when the per-block `@setRuntimeSafety(false)` opt-out has been applied.

### Alternative Backends

For targets where LLVM compilation speed is a concern (particularly during development), Zig provides non-LLVM backends:

- **x86_64 backend** (`src/arch/x86_64/`): direct x86-64 machine code generation from AIR; faster compilation, produces slightly less optimized code; default for `Debug` on x86-64
- **aarch64 backend** (`src/arch/aarch64/`): same for ARM64
- **WebAssembly backend** (`src/arch/wasm/`): direct WASM bytecode generation
- **C backend** (`src/codegen/c.zig`): emits C source; useful for targets without LLVM support, comparable to [Chapter 52 — ClangIR Architecture](../part-08-clangir/ch52-clangir-architecture.md) which also uses C as a portability substrate

The `--backend` flag selects the backend explicitly; in production builds (`ReleaseSafe`, `ReleaseFast`, `ReleaseSmall`), the LLVM backend is always used.

---

## Chapter Summary

- **Three-stage pipeline**: Zig compiles source through ZIR (syntactic, content-addressed) → AIR (typed, monomorphized, comptime-resolved) → LLVM IR (or native backends). Comptime evaluation is complete by the end of Sema; LLVM IR generation is straightforward lowering.

- **ZIR enables incremental caching and IDE tooling**: ZIR is hashed per-module; unchanged ZIR units skip Sema. ZLS uses ZIR for real-time analysis without full compilation.

- **AIR retains Zig semantics**: AIR is a typed SSA IR with Zig-level types (`?T`, `E!T`, `union(enum)`) preserved until lowering. Monomorphization produces distinct AIR functions per comptime argument set.

- **Comptime is not a separate language**: The same Zig code runs at comptime and runtime. Comptime evaluation occurs inside Sema, not in a separate VM. `comptime T: type` parameters enable generic functions; `@typeInfo`, `@typeName`, and `inline for` enable reflection and code generation over types.

- **Error unions are zero-cost**: `E!T` is a `{ u16, T }` struct with error code 0 meaning "ok". Error return traces are a lightweight ring buffer of return addresses, cheaper than full stack traces.

- **Optional types map to `{ T, i1 }`**: Optional pointers use null-pointer optimization (no extra tag bit). Tagged unions become `{ tag, [N x i8] }` payload buffers.

- **Four optimization modes**: `Debug` (safety, no opt), `ReleaseSafe` (safety + O2), `ReleaseFast` (no safety, O3), `ReleaseSmall` (no safety, Oz). Runtime safety checks use LLVM's overflow intrinsics and conditional branches to `@panic`. Per-block opt-out via `@setRuntimeSafety(false)`.

- **`zig cc` is a hermetic C cross-compiler**: Zig bundles Clang, LLD, musl libc source, and platform SDK wrappers. No host sysroot is needed. Used in production by Bun (JavaScriptCore cross-compilation) and TigerBeetle.

- **`build.zig` is a Zig program**: Build logic uses `std.Build` APIs in ordinary Zig code. Content-addressed caching, lazy dependency evaluation, and direct C source file integration are first-class features.

- **LLVM optimization and sanitizers**: Zig uses LLVM's new pass manager. Sanitizers (ASan, UBSan, TSan) work through LLVM instrumentation; Zig's own safety checks in Debug mode already catch most UB in pure Zig code.

---

**References**

- Zig Language Reference (master): [https://ziglang.org/documentation/master/](https://ziglang.org/documentation/master/)
- Andrew Kelley, "The Zig Build System" (LLVM Dev Meeting 2022): [https://llvm.org/devmtg/2022-11/slides/](https://llvm.org/devmtg/2022-11/slides/)
- Andrew Kelley, "A Practical Guide to Applying Data-Oriented Design" (Handmade Seattle 2023): [https://vimeo.com/649009599](https://vimeo.com/649009599)
- Loris Cro, "What is Zig's Comptime?": [https://kristoff.it/blog/what-is-zig-comptime/](https://kristoff.it/blog/what-is-zig-comptime/)
- Zig source — ZIR instruction set: [`src/Zir.zig`](https://github.com/ziglang/zig/blob/master/src/Zir.zig)
- Zig source — AIR instruction set: [`src/Air.zig`](https://github.com/ziglang/zig/blob/master/src/Air.zig)
- Zig source — LLVM codegen: [`src/codegen/llvm.zig`](https://github.com/ziglang/zig/blob/master/src/codegen/llvm.zig)
- Zig source — Sema (comptime + type analysis): [`src/Sema.zig`](https://github.com/ziglang/zig/blob/master/src/Sema.zig)
- TigerBeetle (production Zig): [https://tigerbeetle.com](https://tigerbeetle.com)
- Bun (uses Zig cc for JavaScriptCore): [https://bun.sh](https://bun.sh)

---
*@copyright jreuben11*
