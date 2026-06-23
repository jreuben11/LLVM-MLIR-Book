# Chapter 233 — Emscripten: C/C++ to WebAssembly via LLVM

*Part XXVIII — Language Ecosystems and Engineering Practice*

Emscripten is the standard toolchain for compiling C and C++ to WebAssembly, built directly on LLVM's Clang frontend and Binaryen post-processor. Where Chapter 106 covered WebAssembly as a compilation target in the abstract, this chapter focuses on Emscripten's concrete implementation: how it maps C's memory model to a linear heap, how `EM_JS` and Embind cross the C++/JavaScript boundary, how Asyncify transforms synchronous code to work with JavaScript's event loop, and how Wasm threads integrate with the browser's SharedArrayBuffer model. Emscripten is the bridge that makes the C/C++ software ecosystem available to the Web without source-level changes.

## Table of Contents

- [233.1 Architecture: emcc, Target Triple, and System Libraries](#2331-architecture-emcc-target-triple-and-system-libraries)
  - [Compilation Pipeline](#compilation-pipeline)
  - [Target Triple](#target-triple)
  - [System Libraries](#system-libraries)
  - [Basic Usage](#basic-usage)
- [233.2 Memory Model: Linear Heap and Growth](#2332-memory-model-linear-heap-and-growth)
  - [Memory Layout](#memory-layout)
  - [Memory Growth](#memory-growth)
  - [Pointer Representation](#pointer-representation)
- [233.3 JavaScript Interop: EM_JS and EM_ASM](#2333-javascript-interop-emjs-and-emasm)
  - [EM_ASM: Inline JavaScript](#emasm-inline-javascript)
  - [EM_JS: Named JavaScript Functions](#emjs-named-javascript-functions)
  - [Calling C from JavaScript](#calling-c-from-javascript)
- [233.4 Embind: C++ Object Bindings](#2334-embind-c-object-bindings)
  - [Binding Declarations](#binding-declarations)
  - [JavaScript Usage](#javascript-usage)
  - [Value Types vs. Class Types](#value-types-vs-class-types)
- [233.5 Asyncify: Synchronous Semantics over an Async Event Loop](#2335-asyncify-synchronous-semantics-over-an-async-event-loop)
  - [How Asyncify Works](#how-asyncify-works)
  - [Asyncify Overhead and Optimization](#asyncify-overhead-and-optimization)
  - [JSPI: JavaScript Promise Integration (Alternative)](#jspi-javascript-promise-integration-alternative)
- [233.6 Ports, SDL2, WebGL, and the Event Loop](#2336-ports-sdl2-webgl-and-the-event-loop)
  - [Emscripten Ports](#emscripten-ports)
  - [Main Loop Adaptation](#main-loop-adaptation)
  - [OpenGL → WebGL Mapping](#opengl-webgl-mapping)
- [233.7 Wasm Threads: SharedArrayBuffer and pthreads](#2337-wasm-threads-sharedarraybuffer-and-pthreads)
  - [Requirements](#requirements)
  - [Compilation](#compilation)
  - [pthread Implementation](#pthread-implementation)
  - [Limitations](#limitations)
- [233.8 Output Pipeline: wasm-opt, Closure Compiler, DWARF, and Source Maps](#2338-output-pipeline-wasm-opt-closure-compiler-dwarf-and-source-maps)
  - [Binaryen wasm-opt](#binaryen-wasm-opt)
  - [Closure Compiler](#closure-compiler)
  - [DWARF Debug Info and Source Maps](#dwarf-debug-info-and-source-maps)
  - [Size Optimization](#size-optimization)
- [Chapter Summary](#chapter-summary)

---

## 233.1 Architecture: emcc, Target Triple, and System Libraries

Emscripten wraps `clang` and `lld` with environment configuration, provides browser-compatible system libraries, and orchestrates Binaryen post-processing into a pipeline that produces `.wasm` + `.js` glue code pairs.

### Compilation Pipeline

```
C/C++ source
    │
    ▼ emcc / em++ (wraps clang)
    │  --target=wasm32-unknown-emscripten
    │  sysroot: emsdk/upstream/emscripten/cache/sysroot/
LLVM IR
    │
    ▼ wasm-ld (LLVM's lld WebAssembly port)
    │  links: musl libc + libc++ + Emscripten runtime stubs
Wasm binary (.wasm)
    │
    ▼ Binaryen wasm-opt (optional, -O2/-O3)
    │  + Asyncify transform (if --asyncify)
Optimized .wasm
    │
    ▼ Emscripten JS glue generator
JavaScript runtime (.js) + HTML wrapper (optional)
```

### Target Triple

```
wasm32-unknown-emscripten
  │      │          └── OS: emscripten (not wasi, not emscripten-emulated POSIX)
  │      └── Vendor: unknown
  └── Architecture: wasm32 (32-bit pointer model)
```

`wasm64` (64-bit pointers, MEMORY64 proposal) uses `wasm64-unknown-emscripten`. Emscripten selects 64-bit mode via `-sMEMORY64=1`.

### System Libraries

Emscripten provides:

| Library | Source |
|---------|--------|
| `libc` | musl 1.2.x, compiled to Wasm |
| `libc++` | LLVM's libc++ |
| `libc++abi` | LLVM's libc++abi |
| `libpthread` | Emscripten pthreads (Wasm threads + Atomics) |
| `libGL`/WebGL | Emscripten OpenGL→WebGL translation |
| `libSDL2` | SDL2 port using canvas/WebGL/Web Audio |

Ports (SDL2, zlib, libpng, etc.) are managed by `embuilder` and cached per-architecture.

### Basic Usage

```bash
# Install emsdk
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk && ./emsdk install latest && ./emsdk activate latest
source ./emsdk_env.sh

# Compile a C program
emcc hello.c -o hello.html
# Produces: hello.wasm, hello.js, hello.html

# Link flags passed to Emscripten settings
emcc main.cpp -sEXPORTED_FUNCTIONS=_main,_my_func \
              -sEXPORTED_RUNTIME_METHODS=ccall,cwrap \
              -O2 -o output.js
```

## 233.2 Memory Model: Linear Heap and Growth

WebAssembly's memory model is a linear byte array (`WebAssembly.Memory`). Emscripten maps C's heap onto this array.

### Memory Layout

```
┌──────────┬───────────────┬──────────────────┬───────────────┐
│  Stack   │  Global data  │       Heap        │    ...unused  │
│ (grows ↓)│  (.data/.bss) │   (malloc arena)  │               │
└──────────┴───────────────┴──────────────────┴───────────────┘
0         STACK_BASE      HEAP_BASE           TOTAL_MEMORY
```

- Stack: statically sized (default 64KB, `-sSTACK_SIZE=N`)
- Global data: `.data` and `.bss` segments from the Wasm binary
- Heap: dlmalloc (default), emmalloc (smaller), or user-provided allocator

### Memory Growth

```bash
# Fixed 256 MB
emcc -sTOTAL_MEMORY=268435456 -o output.js main.c

# Dynamically growable (calls memory.grow when needed)
emcc -sALLOW_MEMORY_GROWTH=1 -o output.js main.c

# 64-bit address space (MEMORY64 proposal)
emcc -sMEMORY64=1 -o output.js main.c  # wasm64
```

`ALLOW_MEMORY_GROWTH` can disable Safari's JIT optimizations for the Wasm module (the browser must assume the backing ArrayBuffer can be detached and replaced). Use `-sINITIAL_MEMORY` to over-provision if growth is anticipated.

### Pointer Representation

In wasm32 mode, all pointers are 32-bit integers—C `sizeof(void*)` is 4. In JavaScript, pointers appear as numbers. The full C heap is accessible from JavaScript as `HEAPU8`, `HEAP32`, `HEAPF64`, etc.:

```javascript
// Read a C int at address ptr
const value = Module.HEAP32[ptr >> 2];

// Write a C double at address ptr
Module.HEAPF64[ptr >> 3] = 3.14159;
```

## 233.3 JavaScript Interop: EM_JS and EM_ASM

Emscripten provides two mechanisms for embedding JavaScript directly in C/C++ source.

### EM_ASM: Inline JavaScript

`EM_ASM` executes a JavaScript snippet inline. Arguments are passed as `$0`, `$1`, ...; the return value comes from a JavaScript expression:

```c
#include <emscripten.h>

void alert_from_c(const char *msg) {
    EM_ASM({
        alert(UTF8ToString($0));
    }, msg);
}

int get_window_width(void) {
    return EM_ASM_INT({ return window.innerWidth; });
}
```

`EM_ASM` code is embedded verbatim in the JS glue and evaluated at runtime. The `{...}` block is a JavaScript function body. `EM_ASM_INT`, `EM_ASM_DOUBLE` variants specify return types.

### EM_JS: Named JavaScript Functions

`EM_JS` declares a C function whose body is implemented in JavaScript—cleaner than `EM_ASM` for complex JS with proper names and IDE support:

```c
EM_JS(void, js_console_log, (const char *str), {
    console.log(UTF8ToString(str));
});

EM_JS(int, js_fetch_sync, (const char *url), {
    // NOTE: synchronous XHR—use Asyncify for proper async
    var xhr = new XMLHttpRequest();
    xhr.open("GET", UTF8ToString(url), false);
    xhr.send();
    return xhr.status;
});
```

`EM_JS` functions are available as regular C functions; the linker resolves them from the generated JS module.

### Calling C from JavaScript

`ccall` and `cwrap` call exported C functions from JavaScript with automatic type marshalling:

```javascript
// Direct call
const result = Module.ccall(
    'my_c_function',   // C function name (without leading _)
    'number',          // return type: 'number', 'string', 'array', null
    ['number', 'string'], // argument types
    [42, "hello"]      // argument values
);

// Wrap for repeated calls
const myFunc = Module.cwrap('my_c_function', 'number', ['number', 'string']);
const r1 = myFunc(1, "a");
const r2 = myFunc(2, "b");
```

String arguments are automatically copied to/from the Wasm heap. For `'array'` type, pass a `Uint8Array`—Emscripten handles the copy.

## 233.4 Embind: C++ Object Bindings

Embind provides a C++ API for exporting classes, functions, and enumerations to JavaScript with rich type semantics—no manual pointer management required.

### Binding Declarations

```cpp
#include <emscripten/bind.h>
using namespace emscripten;

class Vector3 {
public:
    float x, y, z;
    Vector3(float x, float y, float z) : x(x), y(y), z(z) {}
    float length() const { return std::sqrt(x*x + y*y + z*z); }
    Vector3 add(const Vector3& other) const {
        return Vector3(x + other.x, y + other.y, z + other.z);
    }
    static Vector3 zero() { return Vector3(0, 0, 0); }
};

EMSCRIPTEN_BINDINGS(vector3_module) {
    class_<Vector3>("Vector3")
        .constructor<float, float, float>()
        .property("x", &Vector3::x)
        .property("y", &Vector3::y)
        .property("z", &Vector3::z)
        .function("length", &Vector3::length)
        .function("add", &Vector3::add)
        .class_function("zero", &Vector3::zero);

    function("createVector", &createVector);  // free function

    enum_<Direction>("Direction")
        .value("North", Direction::North)
        .value("South", Direction::South);
}
```

Compile with `--bind`:

```bash
em++ vector3.cpp --bind -o vector3.js
```

### JavaScript Usage

```javascript
const Module = await import('./vector3.js');
const v1 = new Module.Vector3(1, 2, 3);
const v2 = new Module.Vector3(4, 5, 6);
const v3 = v1.add(v2);
console.log(v3.length());  // 10.392...
v1.delete();  // explicit memory management for heap-allocated objects
v2.delete();
v3.delete();
```

Embind objects allocated on the Wasm heap require explicit `delete()` calls. For automatic cleanup, use `emscripten::val` smart references or the `using` wrapper in JavaScript (`EmscriptenModule.destroy(obj)`).

### Value Types vs. Class Types

- `value_object<T>`: copies T by value into/out of Wasm heap; no `delete()` needed
- `class_<T>`: heap-allocated T with reference semantics; requires `delete()`
- `register_vector<T>`: wraps `std::vector<T>` with JavaScript array protocol

```cpp
EMSCRIPTEN_BINDINGS(color_module) {
    value_object<Color>("Color")
        .field("r", &Color::r)
        .field("g", &Color::g)
        .field("b", &Color::b);
}
```

Source: [`system/include/emscripten/bind.h`](https://github.com/emscripten-core/emscripten/blob/main/system/include/emscripten/bind.h)

## 233.5 Asyncify: Synchronous Semantics over an Async Event Loop

JavaScript's event loop never blocks—`sleep()`, `pthread_join()`, and synchronous I/O cannot be implemented as blocking calls in the browser's main thread. Asyncify transforms compiled Wasm to support these patterns by unrolling the call stack to a heap-allocated continuation.

### How Asyncify Works

Asyncify is a Binaryen pass (`wasm-opt --asyncify`) that instruments each function in the call stack of a "blocking" operation:

1. **Unwind phase**: When a blocking call is encountered, the function saves its local state (locals, PC position) to a heap-allocated `asyncify_data` buffer and returns to JavaScript
2. JavaScript performs the async operation (e.g., `fetch()`, timer, I/O) and gets the result
3. **Rewind phase**: When JavaScript calls back into Wasm, Asyncify restores the saved state and resumes execution from the save point

```c
#include <emscripten.h>

// sleep() becomes asynchronous via Asyncify
void do_work(void) {
    printf("start\n");
    emscripten_sleep(1000);  // suspends for 1 second, returns to JS event loop
    printf("after sleep\n");
}
```

```bash
emcc asyncify_example.c -sASYNCIFY -o out.js
```

### Asyncify Overhead and Optimization

Without filtering, Asyncify instruments every function in the module, which can increase binary size by 2–5×. Use `ASYNCIFY_IMPORTS` to specify which imports can suspend, and `ASYNCIFY_ONLY`/`ASYNCIFY_REMOVE` to limit instrumentation:

```bash
emcc -sASYNCIFY \
     -sASYNCIFY_IMPORTS='["my_blocking_import"]' \
     -sASYNCIFY_ONLY='["do_work", "helper_func"]' \
     main.c -o out.js
```

### JSPI: JavaScript Promise Integration (Alternative)

The WebAssembly [JavaScript Promise Integration (JSPI)](https://github.com/WebAssembly/js-promise-integration) proposal (now in Origin Trial in Chrome) provides a first-class mechanism for suspending Wasm on import calls that return Promises, without Binaryen instrumentation:

```javascript
// JSPI wraps a Wasm export so that JS can await it
const { wrappedExport } = WebAssembly.promising(instance.exports.do_work);
await wrappedExport();
```

JSPI is lower overhead than Asyncify (no stack copying) and is available via `-sJSPI=1` in recent Emscripten. The two approaches are complementary: JSPI for new code targeting Chrome, Asyncify for maximum browser compatibility.

## 233.6 Ports, SDL2, WebGL, and the Event Loop

### Emscripten Ports

Many C/C++ libraries are available as Emscripten ports, compiled to Wasm on demand:

```bash
embuilder build sdl2 sdl2-image sdl2-ttf zlib libpng freetype harfbuzz
emcc main.c -sUSE_SDL=2 -sUSE_SDL_IMAGE=2 -sUSE_ZLIB=1 -o out.html
```

The port system downloads, patches, and caches libraries in `~/.emscripten_cache/`. Ports expose the same header API as native builds—porting an SDL2 application typically requires no source changes.

### Main Loop Adaptation

C applications traditionally have an infinite `while(1)` main loop that would block JavaScript's event loop. Emscripten requires this to be replaced with a callback model:

```c
#include <SDL2/SDL.h>
#include <emscripten.h>

void main_loop_iteration(void *arg) {
    SDL_Event event;
    while (SDL_PollEvent(&event)) {
        if (event.type == SDL_QUIT) emscripten_cancel_main_loop();
    }
    render_frame();
}

int main(void) {
    SDL_Init(SDL_INIT_VIDEO);
    SDL_Window *window = SDL_CreateWindow("Demo", 0, 0, 800, 600, 0);
    setup_renderer(window);
    emscripten_set_main_loop_arg(main_loop_iteration, NULL, 0, 1);
    // The '1' causes main to return immediately; the loop runs via requestAnimationFrame
    return 0;
}
```

`emscripten_set_main_loop` registers a callback driven by `requestAnimationFrame`, giving native 60fps rendering without blocking.

### OpenGL → WebGL Mapping

Emscripten provides `libGL`, a source-compatible OpenGL ES 2.0/3.0 implementation that translates OpenGL calls to WebGL:

```bash
emcc gl_demo.c -lGL -sUSE_WEBGL2=1 -sFULL_ES3=1 -o demo.html
```

WebGL 2 corresponds to OpenGL ES 3.0; most GL ES 3.0 code compiles without modification. `EM_WEBGL_POWER_PREFERENCE` and `EM_WEBGL_ANTIALIAS` control WebGL context attributes.

## 233.7 Wasm Threads: SharedArrayBuffer and pthreads

Emscripten supports POSIX threads via WebAssembly threads (the `threads` proposal: shared memory + atomics) with `SharedArrayBuffer` as the backing store.

### Requirements

Wasm threads require the browser to set two security headers (COOP/COEP) to enable `SharedArrayBuffer`:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

These headers are required because `SharedArrayBuffer` was disabled after Spectre/Meltdown. Without them, `SharedArrayBuffer` is `undefined` and threaded Wasm will not work.

### Compilation

```bash
emcc threaded_main.cpp \
  -sUSE_PTHREADS=1 \
  -sPTHREAD_POOL_SIZE=4 \
  -O2 \
  -o output.js
```

This produces `output.js`, `output.wasm`, and `output.worker.js`. The worker script is the Web Worker entry point for each thread.

### pthread Implementation

```c
#include <pthread.h>
#include <emscripten.h>

typedef struct { int start; int end; double *result; } WorkArgs;

void *compute_range(void *arg) {
    WorkArgs *a = (WorkArgs *)arg;
    double sum = 0.0;
    for (int i = a->start; i < a->end; i++) sum += 1.0 / i;
    *a->result = sum;
    return NULL;
}

int main(void) {
    pthread_t t1, t2;
    double r1, r2;
    WorkArgs a1 = {1, 500000, &r1};
    WorkArgs a2 = {500000, 1000000, &r2};
    pthread_create(&t1, NULL, compute_range, &a1);
    pthread_create(&t2, NULL, compute_range, &a2);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("sum = %f\n", r1 + r2);
}
```

Under the hood, `pthread_create` posts a message to a Web Worker from the pre-allocated pool (`PTHREAD_POOL_SIZE`). Shared memory (the Wasm `memory` object with `shared=true`) backs the `pthread_t` synchronization primitives using `Atomics.wait` and `Atomics.notify`.

### Limitations

- `Atomics.wait` cannot block the main thread (the browser's event loop thread)—use `PTHREAD_POOL_SIZE_STRICT=2` to detect main-thread blocking attempts at development time
- Workers cannot directly access the DOM; use `emscripten_sync_run_in_main_thread_*` or `Atomics`-based message passing
- SharedArrayBuffer requires COOP/COEP headers; this complicates CDN delivery of multi-threaded apps

## 233.8 Output Pipeline: wasm-opt, Closure Compiler, DWARF, and Source Maps

### Binaryen wasm-opt

After linking, Emscripten optionally runs Binaryen's `wasm-opt` for additional Wasm-level optimization:

```bash
emcc -O3 main.c -o out.js   # runs wasm-opt -O3 internally
# Equivalent manual step:
wasm-opt -O3 --enable-simd out.wasm -o out.opt.wasm
```

Key `wasm-opt` passes useful post-Emscripten:

| Pass | Effect |
|------|--------|
| `-O3` | Full optimization (inlining, DCE, GVN) |
| `--asyncify` | Stack-unrolling transform for Asyncify |
| `--strip-debug` | Remove DWARF sections |
| `--enable-simd` | Allow 128-bit SIMD lowering |
| `--minify-imports-and-exports` | Shorten import/export names |

### Closure Compiler

JavaScript glue code can be minified with Google Closure Compiler:

```bash
emcc -O2 --closure 1 main.c -o out.js
```

`--closure 2` enables advanced optimizations (property renaming); this requires Embind-generated code to use `CLOSURE_COMPILER_FRIENDLY` annotations.

### DWARF Debug Info and Source Maps

For browser-side debugging with original source file + line numbers:

```bash
# Emit DWARF sections in the Wasm binary
emcc -g main.c -o out.js   # DWARF embedded in .wasm

# Or source maps (Chrome DevTools-style)
emcc -gsource-map main.c -o out.js
# Produces out.wasm.map; Chrome DevTools maps Wasm bytes → source
```

Chrome DevTools can step through the original C/C++ source if it can fetch the source files (or they are embedded via `--embed-file`). DWARF in Wasm is supported by the [C/C++ DevTools Support extension](https://goo.gle/wasm-debugging-extension).

### Size Optimization

| Technique | Flag | Typical saving |
|-----------|------|---------------|
| Wasm optimization | `-O2` / `-O3` | 20–40% |
| Dead code elimination | `-sEXPORTED_FUNCTIONS` | 10–30% |
| LTO | `-flto` | 5–15% |
| wasm-opt | `-O3` post-link | 5–10% additional |
| Closure JS minification | `--closure 1` | 50–70% of JS glue |
| Wasm binary compression (gzip/brotli) | Server-side | 60–80% transfer |

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **JSPI standardization and broad adoption**: The WebAssembly JavaScript Promise Integration proposal is advancing through W3C standardization after its Chrome Origin Trial; Emscripten's `-sJSPI=1` flag is expected to stabilize and `-sASYNCIFY` will be positioned as the legacy fallback. Track [WebAssembly/js-promise-integration](https://github.com/WebAssembly/js-promise-integration) for spec finalization.
- **MEMORY64 (wasm64) production readiness**: The wasm64 (`-sMEMORY64=1`) path is nearing stable status in Emscripten as browser engines (V8, SpiderMonkey) complete their MEMORY64 implementations; expect production-ready wasm64 support enabling >4 GB heaps for large scientific and media workloads by late 2026.
- **Binaryen stack-switching integration**: Binaryen's stack-switching transform (the WebAssembly [stack-switching proposal](https://github.com/WebAssembly/stack-switching)) is being prototyped as a lower-overhead alternative to Asyncify's unwind/rewind model; early Emscripten integration (`--stack-switching`) targets experimental use in 2026.
- **Embind ES module output and tree-shaking**: Emscripten's `--bind` output is being refactored to emit native ES modules with named exports, enabling bundlers (Vite, Webpack, Rollup) to tree-shake unused Embind bindings; the [emscripten#21xxx RFC thread](https://github.com/emscripten-core/emscripten/issues) tracks this effort.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **WebAssembly GC interop from C++**: As the [Wasm GC proposal](https://github.com/WebAssembly/gc) (now at Phase 4) reaches full browser support, Emscripten will need to integrate Clang's C++ object model with the Wasm GC type system to avoid double-heap overhead when interoperating with JavaScript GC objects—expect new Embind APIs targeting `externref` and `anyref` types.
- **Component Model adoption replacing JS glue**: The WebAssembly [Component Model](https://github.com/WebAssembly/component-model) and its WIT (Wasm Interface Types) IDL aim to replace hand-written JS glue code (EM_JS, ccall/cwrap) with a standardized ABI; Emscripten will gain a `--component` output mode as the toolchain (`wasm-tools`, `wit-bindgen`) matures.
- **SIMD 128 + Relaxed SIMD optimization integration**: Clang's autovectorizer already targets Wasm SIMD 128; with Relaxed SIMD (Phase 4) widely deployed, Emscripten and wasm-opt will gain backend passes that exploit `relaxed_madd`, `i8x16.swizzle`, and related instructions for DSP, ML inference, and image processing code.
- **WASI Preview 2 / WASI P3 and Emscripten WASI mode**: WASI Preview 2 (socket, HTTP, filesystem components) standardizes APIs that Emscripten currently implements with browser-specific shims; a unified `emcc --target wasi` mode that supports both browser and server runtimes (Wasmtime, WasmEdge) is on the roadmap, collapsing the wasm32-unknown-emscripten / wasm32-wasi split.

### 5-Year Horizon (Long-Term, by ~2031)

- **Wasm native debugging without extensions**: The DWARF-in-Wasm format and the Chrome DevTools extension (`wasm-debugging-extension`) are expected to be fully integrated into all major browser DevTools natively, eliminating the need for the separate extension and making C/C++ source-level debugging in the browser as seamless as JavaScript debugging.
- **Multi-memory and shared-everything threads**: The [WebAssembly multi-memory](https://github.com/WebAssembly/multi-memory) and [shared-everything threads](https://github.com/WebAssembly/shared-everything-threads) proposals, if adopted, will allow Emscripten to map separate C heaps, stack segments, and TLS regions onto distinct Wasm memories, eliminating the single-linear-memory constraint and enabling safer isolation between components in the same module.
- **C++ modules and binary interface standardization**: LLVM's work on C++20/C++ modules (BMI files) combined with the Wasm Component Model could yield a stable binary interface for C++ libraries distributed as `.wasm` components—analogous to shared libraries—shareable across applications without recompilation, ending the "recompile everything from source" requirement of the current Emscripten port system.

---

## Chapter Summary

- Emscripten compiles C/C++ to WebAssembly via `clang --target=wasm32-unknown-emscripten`, `wasm-ld`, and Binaryen post-processing; the sysroot includes musl libc, libc++, and browser-compatible ports of SDL2/OpenGL
- **Memory model**: a linear `WebAssembly.Memory` mapped as HEAPU8/HEAP32/HEAPF64; 32-bit pointers in wasm32 mode; `ALLOW_MEMORY_GROWTH` enables runtime expansion
- **EM_ASM** embeds inline JavaScript snippets; **EM_JS** declares named C functions implemented in JavaScript; both integrate with the same Wasm module JS glue
- **Embind** (`EMSCRIPTEN_BINDINGS`) exports C++ classes, functions, and enumerations to JavaScript with typed semantics; `class_<T>` requires explicit `delete()`, `value_object<T>` copies by value
- **Asyncify** (Binaryen pass) transforms Wasm to unwind/rewind the call stack around blocking operations, enabling `sleep()`, synchronous I/O, and `pthread_join` in the browser; JSPI is the lower-overhead upcoming alternative
- **Wasm threads** map POSIX pthreads to Web Workers sharing a `WebAssembly.Memory` with `shared=true`; requires `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` headers
- **wasm-opt** post-processes the Wasm binary with Binaryen optimization passes; Closure Compiler minifies the JS glue; `-g`/`-gsource-map` emit DWARF or source maps for browser debugging
- Emscripten makes the vast majority of portable C/C++ code available to Web platforms with minimal or no source changes, while exposing JavaScript interop APIs at multiple levels of abstraction

**Cross-references**: [Chapter 106 — WebAssembly and BPF](ch106-webassembly-and-bpf.md) | [Chapter 230 — Cranelift: A Lightweight JIT for WebAssembly and Rust](../part-28-language-ecosystems/ch230-cranelift-lightweight-jit.md) | [Chapter 103 — AMDGPU and the ROCm Path](ch103-amdgpu-and-the-rocm-path.md)

**References**:
- [Emscripten Documentation](https://emscripten.org/docs/)
- [Emscripten GitHub Repository](https://github.com/emscripten-core/emscripten)
- [WebAssembly Threads Proposal](https://github.com/WebAssembly/threads)
- [Asyncify: Blocking Code on the Web](https://kripken.github.io/blog/wasm/2019/07/16/asyncify.html)
- [Binaryen wasm-opt Reference](https://github.com/WebAssembly/binaryen/blob/main/README.md)
- [JSPI Proposal](https://github.com/WebAssembly/js-promise-integration)
- [Embind Documentation](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html)
