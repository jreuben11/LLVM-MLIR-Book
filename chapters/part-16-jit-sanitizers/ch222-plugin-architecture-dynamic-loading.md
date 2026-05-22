# Chapter 222 — Plugin Architecture, Dynamic Loading, and ABI Stability

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Every major software system eventually needs to load code it did not contain at build time: audio workstations load synthesizer plugins, compilers load backend targets, databases load procedural language extensions, and games load mod content. Dynamic loading — the OS-level mechanism for mapping a shared library into a running process and resolving its symbols — is the foundational technology beneath all of these use cases. Unlike the JIT approaches in the preceding chapters, dynamic loading works at the granularity of shared objects (`.so`, `.dll`, `.dylib`) compiled separately and loaded at runtime via a stable OS interface. Getting this right requires understanding the POSIX `dlopen`/`dlsym` API, ELF ABI constraints, C++ boundary rules, soname versioning, and platform-specific behavior on Windows and macOS. This chapter covers all of these, then examines real-world plugin architectures and closes with LLVM's own `llvm::sys::DynamicLibrary` abstraction.

---

## Table of Contents

- [222.1 POSIX Dynamic Loading Fundamentals](#2221-posix-dynamic-loading-fundamentals)
  - [dlopen: Loading a Shared Library](#dlopen-loading-a-shared-library)
  - [dlsym: Looking Up a Symbol](#dlsym-looking-up-a-symbol)
  - [dlclose and Reference Counting](#dlclose-and-reference-counting)
  - [Runtime Linker Search Order](#runtime-linker-search-order)
  - [dladdr and dl_iterate_phdr](#dladdr-and-dliteratephdr)
- [222.2 ABI Stability as a Contract](#2222-abi-stability-as-a-contract)
  - [Why C Linkage is the Plugin Lingua Franca](#why-c-linkage-is-the-plugin-lingua-franca)
  - [Soname Versioning](#soname-versioning)
  - [Symbol Versioning with Linker Scripts](#symbol-versioning-with-linker-scripts)
  - [The Add-Only Compatibility Rule](#the-add-only-compatibility-rule)
- [222.3 Type-Safe C++ Plugin Interface Patterns](#2223-type-safe-c-plugin-interface-patterns)
  - [Abstract Factory Pattern](#abstract-factory-pattern)
  - [NVI Version Negotiation](#nvi-version-negotiation)
  - [What Cannot Cross Plugin Boundaries](#what-cannot-cross-plugin-boundaries)
- [222.4 Plugin Discovery and Registration](#2224-plugin-discovery-and-registration)
  - [Compile-Time Registration via Constructors](#compile-time-registration-via-constructors)
  - [Runtime Discovery via Directory Scan](#runtime-discovery-via-directory-scan)
  - [Manifest-Based Discovery](#manifest-based-discovery)
- [222.5 Windows and macOS Equivalents](#2225-windows-and-macos-equivalents)
  - [Windows: LoadLibrary/GetProcAddress](#windows-loadlibrarygetprocaddress)
  - [Windows vs POSIX Comparison](#windows-vs-posix-comparison)
  - [macOS: dlopen with Gatekeeper Constraints](#macos-dlopen-with-gatekeeper-constraints)
- [222.6 llvm::sys::DynamicLibrary](#2226-llvmsysdynamiclibrary)
  - [getPermanentLibrary](#getpermanentlibrary)
  - [DynamicLibrarySearchGenerator](#dynamiclibrarysearchgenerator)
  - [Cross-Platform Notes](#cross-platform-notes)
- [222.7 Real-World Plugin Architectures](#2227-real-world-plugin-architectures)
  - [GCC Plugin API](#gcc-plugin-api)
  - [LLVM's Own Backend Plugins](#llvms-own-backend-plugins)
  - [VST3 Audio Plugin Interface](#vst3-audio-plugin-interface)
  - [LV2 Audio Plugins](#lv2-audio-plugins)
  - [Game Mod Systems](#game-mod-systems)
- [222.8 Security Model and Mitigations](#2228-security-model-and-mitigations)
  - [dlopen as Arbitrary Code Execution](#dlopen-as-arbitrary-code-execution)
  - [Attack Vectors](#attack-vectors)
  - [Mitigations](#mitigations)
  - [Cryptographic Signature Verification](#cryptographic-signature-verification)
- [222.9 Interaction with ORC JIT and Self-Modification](#2229-interaction-with-orc-jit-and-self-modification)
  - [Combined Architecture: Stable ABI + Mutable Implementation](#combined-architecture-stable-abi-mutable-implementation)
  - [DynamicLibrarySearchGenerator in the Combined Model](#dynamiclibrarysearchgenerator-in-the-combined-model)
- [Chapter Summary](#chapter-summary)

---

## 222.1 POSIX Dynamic Loading Fundamentals

The POSIX dynamic loading API consists of four functions declared in `<dlfcn.h>`:

```c
void* dlopen(const char* filename, int flags);
void* dlsym(void* handle, const char* symbol);
int   dlclose(void* handle);
char* dlerror(void);
```

### dlopen: Loading a Shared Library

```c
#include <dlfcn.h>

void* handle = dlopen("/path/to/libplugin.so", RTLD_NOW | RTLD_LOCAL);
if (!handle) {
    fprintf(stderr, "dlopen failed: %s\n", dlerror());
    return;
}
```

#### RTLD_NOW vs RTLD_LAZY

| Flag | Behavior | Use when |
|------|----------|----------|
| `RTLD_NOW` | Resolve all undefined symbols immediately; fail if any missing | Production: fail-fast on missing dependencies |
| `RTLD_LAZY` | Resolve symbols on first use | Development/optional features; deferred error |

`RTLD_NOW` is strongly preferred for production plugin loading: a missing symbol causes `dlopen` to fail with a clear error message rather than a segfault at the first call to the undefined symbol.

#### RTLD_LOCAL vs RTLD_GLOBAL

| Flag | Behavior | Use when |
|------|----------|----------|
| `RTLD_LOCAL` | Library symbols are not exposed to subsequently loaded libraries | Default; hermetic plugins that should not affect global symbol table |
| `RTLD_GLOBAL` | Library symbols are added to the global symbol table | Plugin that provides symbols needed by other plugins; use carefully |

`RTLD_GLOBAL` can cause symbol collisions if multiple plugins define the same symbol name. `RTLD_LOCAL` is the safe default; plugins that need to share symbols should link against a common shared library rather than relying on `RTLD_GLOBAL`.

#### RTLD_DEEPBIND (Linux glibc extension)

```c
// Plugin uses its own copy of a symbol, not the host's copy
void* handle = dlopen("plugin.so", RTLD_NOW | RTLD_LOCAL | RTLD_DEEPBIND);
```

`RTLD_DEEPBIND` makes the plugin search its own symbol table before the global table. Useful when the plugin embeds a different version of a library (e.g., libstdc++) than the host. Not available on macOS.

### dlsym: Looking Up a Symbol

```c
typedef int (*PluginInitFn)(void);

// Look up symbol in a specific library
PluginInitFn init = (PluginInitFn)dlsym(handle, "plugin_init");

// Look up symbol in any loaded library (RTLD_DEFAULT)
void* fn = dlsym(RTLD_DEFAULT, "some_symbol");

// Look up next symbol in search order (RTLD_NEXT) — used in interpositioning
void* orig_malloc = dlsym(RTLD_NEXT, "malloc");
```

`dlsym` returns `NULL` both for "symbol not found" and for a symbol whose value is genuinely `NULL`. Always call `dlerror()` after `dlsym`:

```c
dlerror();  // clear any pending error
void* sym = dlsym(handle, "my_func");
const char* err = dlerror();
if (err) {
    fprintf(stderr, "dlsym failed: %s\n", err);
}
```

### dlclose and Reference Counting

The dynamic linker maintains a reference count per library. `dlopen` increments it; `dlclose` decrements it. The library is unmapped only when the count reaches zero and no other loaded library depends on it.

```c
dlclose(handle);  // decrement; may or may not unmap
```

Consequence: calling `dlclose` does not immediately invalidate function pointers obtained from the library. Those pointers remain valid until the library is actually unmapped. However, calling a function pointer after the library is unmapped is undefined behavior.

### Runtime Linker Search Order

When `dlopen("libplugin.so", ...)` is called without an absolute path, the runtime linker searches in order:

1. `DT_RPATH` of the executable (embedded at link time; deprecated in favor of `runpath`)
2. `LD_LIBRARY_PATH` environment variable (colon-separated directory list)
3. `DT_RUNPATH` of the executable (embedded; respects `LD_LIBRARY_PATH`)
4. `/etc/ld.so.cache` (precomputed from `/etc/ld.so.conf`)
5. `/lib`, `/usr/lib` (default system paths)

Best practice: embed an absolute path in `dlopen` for production plugin loaders, or use a restricted `DT_RUNPATH` ($ORIGIN-relative). Never rely on `LD_LIBRARY_PATH` in production.

```c
// $ORIGIN-relative rpath: looks for plugins in the same directory as the binary
// Set at link time: -Wl,-rpath,'$ORIGIN/plugins'
dlopen("libplugin_foo.so", RTLD_NOW | RTLD_LOCAL);
// Finds: /path/to/binary/../plugins/libplugin_foo.so
```

### dladdr and dl_iterate_phdr

```c
// Reverse lookup: from address to symbol name and library
Dl_info info;
if (dladdr((void*)&some_function, &info)) {
    printf("Found in: %s\n", info.dli_fname);
    printf("Symbol: %s at %p\n", info.dli_sname, info.dli_saddr);
}

// Enumerate all loaded shared objects
dl_iterate_phdr([](dl_phdr_info* info, size_t size, void* data) -> int {
    printf("Library: %s at %p\n", info->dlpi_name, (void*)info->dlpi_addr);
    return 0;
}, nullptr);
```

`dladdr` is essential for debugging: given a function pointer from a plugin, it returns the shared library name and the symbol's load address. `dl_iterate_phdr` enumerates all loaded ELF objects, useful for finding all currently active plugins.

---

## 222.2 ABI Stability as a Contract

The fundamental problem of plugin interfaces: the host binary and the plugin binary are compiled at different times, possibly with different compilers, different compiler versions, and different optimization flags. The ABI — the binary contract between the two — must remain stable across all these variations.

### Why C Linkage is the Plugin Lingua Franca

C++ name mangling is compiler-specific and version-specific. A function `void Foo::bar(int)` in GCC 12 produces a different mangled name than in GCC 14 or Clang 22. Calling a function through its mangled name across a compiler boundary produces a linker error at best and a stack corruption at worst.

C linkage (`extern "C"`) disables name mangling and uses the platform C ABI:

```cpp
// plugin.h — the stable public interface
#ifdef __cplusplus
extern "C" {
#endif

typedef void* PluginHandle;
PluginHandle plugin_create(void);
void         plugin_destroy(PluginHandle h);
int          plugin_process(PluginHandle h, const float* in, float* out, int n);

#ifdef __cplusplus
}
#endif
```

The implementation (inside the plugin) uses full C++ internally:

```cpp
// plugin_impl.cpp
#include "plugin.h"
#include <memory>

struct PluginImpl {
    // Full C++ internals here — templates, RAII, virtual dispatch
    float gain = 1.0f;
};

extern "C" {
PluginHandle plugin_create() {
    return new PluginImpl;
}
void plugin_destroy(PluginHandle h) {
    delete static_cast<PluginImpl*>(h);
}
int plugin_process(PluginHandle h, const float* in, float* out, int n) {
    auto* impl = static_cast<PluginImpl*>(h);
    for (int i = 0; i < n; ++i) out[i] = in[i] * impl->gain;
    return 0;
}
}
```

The host calls only the C-linkage functions and never touches `PluginImpl` directly. The plugin can change its internal C++ structure freely without breaking the host.

### Soname Versioning

A *soname* (`DT_SONAME` ELF dynamic entry) identifies the library's ABI version:

```bash
# Build with soname
clang++ -shared -fPIC -Wl,-soname,libplugin.so.1 \
    plugin_impl.cpp -o libplugin.so.1.2.3

# Create symlinks
ln -s libplugin.so.1.2.3 libplugin.so.1   # runtime linker uses this
ln -s libplugin.so.1     libplugin.so      # linker uses this at compile time
```

The soname convention:
- **Major version** (`.1`): incompatible ABI change; host must recompile
- **Minor version** (`.1.2`): compatible addition (new symbols); old host still works
- **Patch version** (`.1.2.3`): bug fix; no ABI impact

When the host binary is linked with `-lplugin`, the linker embeds `libplugin.so.1` (the soname) in `DT_NEEDED`. At runtime, the dynamic linker finds `libplugin.so.1`, which resolves to the current patch version.

### Symbol Versioning with Linker Scripts

For fine-grained compatibility, use symbol version scripts:

```
# plugin.map — version script
PLUGIN_API_1 {
    global:
        plugin_create;
        plugin_destroy;
        plugin_process;
    local: *;  /* hide everything else */
};

PLUGIN_API_2 {
    global:
        plugin_process_v2;  /* new in ABI version 2 */
} PLUGIN_API_1;
```

```bash
clang++ -shared -fPIC -Wl,--version-script=plugin.map \
    plugin_impl.cpp -o libplugin.so.2
```

Symbol versioning allows keeping multiple versions of a function in the same library, with the dynamic linker selecting the correct version based on what the caller was compiled against.

### The Add-Only Compatibility Rule

ABI-safe changes (bump minor version):
- Add new `extern "C"` functions
- Add new parameters at the end of a struct (if the struct is always accessed via pointer and size is checked)

ABI-breaking changes (bump major version, require host recompile):
- Remove any `extern "C"` function
- Rename any `extern "C"` function
- Change any function signature (parameter types, return type, calling convention)
- Change the layout of any struct in the public header
- Change the size of any enum used in the public interface

---

## 222.3 Type-Safe C++ Plugin Interface Patterns

Pure C interfaces are safe but ergonomically painful. Several patterns provide C++ ergonomics while maintaining ABI safety.

### Abstract Factory Pattern

The most common pattern: an `extern "C"` factory function returns a pointer to an abstract C++ base class. All interaction goes through the base class's virtual interface.

```cpp
// IPlugin.h — the stable abstract interface
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual int getVersion() const noexcept = 0;
    virtual int process(const float* in, float* out, int n) noexcept = 0;
};

extern "C" {
    // Factory function: the ONLY extern "C" symbol needed
    IPlugin* create_plugin();
    void     destroy_plugin(IPlugin*);
}
```

```cpp
// Host usage:
using CreateFn  = IPlugin*(*)();
using DestroyFn = void(*)(IPlugin*);

void* handle = dlopen("libplugin.so", RTLD_NOW | RTLD_LOCAL);
auto create  = reinterpret_cast<CreateFn>(dlsym(handle, "create_plugin"));
auto destroy = reinterpret_cast<DestroyFn>(dlsym(handle, "destroy_plugin"));

std::unique_ptr<IPlugin, DestroyFn> plugin(create(), destroy);
plugin->process(inputBuf, outputBuf, nSamples);
```

The `std::unique_ptr<IPlugin, DestroyFn>` ensures the plugin is destroyed by the plugin's `destroy_plugin` function (which calls the correct `delete`, using the plugin's allocator), not the host's.

### NVI Version Negotiation

When the interface may evolve across major versions, add a version query method:

```cpp
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual int apiVersion() const noexcept = 0;

    // NVI: public non-virtual calls protected virtual
    int process(const float* in, float* out, int n) noexcept {
        return doProcess(in, out, n);
    }

protected:
    virtual int doProcess(const float* in, float* out, int n) noexcept = 0;
};

// Host queries version before using advanced features:
if (plugin->apiVersion() >= 2) {
    auto* p2 = static_cast<IPluginV2*>(plugin.get());
    p2->processV2(...);
}
```

### What Cannot Cross Plugin Boundaries

The following C++ types are ABI-unsafe across plugin boundaries:

| Type | Problem |
|------|---------|
| `std::string` | ABI changes between libstdc++ and libc++; between debug/release builds |
| `std::vector<T>` | Internal layout (pointer/size/capacity) is ABI-defined but implementation-specific |
| Exceptions | Exception ABI (`__cxa_throw`) requires matching runtime; mixed runtimes = crash |
| `std::shared_ptr<T>` | Reference count layout is ABI-specific |
| `std::function` | Type-erased representation is implementation-defined |
| RTTI (`typeid`, `dynamic_cast`) | Type info objects use symbol addresses; mismatched runtimes = false negatives |

Safe alternatives:
- Pass `const char*` instead of `std::string`
- Pass `T*, size_t` pairs instead of `std::vector<T>`
- Use `std::string_view` / `std::span<T>` (layout-stable read-only views, C++17/20)
- Communicate errors via return codes or out-pointer error structs, not exceptions

---

## 222.4 Plugin Discovery and Registration

### Compile-Time Registration via Constructors

```cpp
// plugin_registry.h
class PluginRegistry {
public:
    static PluginRegistry& instance();
    void registerPlugin(const char* name, IPlugin*(*)());
    IPlugin* create(const char* name);
};

// auto_register.cpp (inside the plugin binary)
static int __attribute__((constructor)) auto_register() {
    PluginRegistry::instance().registerPlugin("my_plugin", &create_plugin);
    return 0;
}
```

The `__attribute__((constructor))` function runs at `dlopen` time (before the handle is returned), registering the plugin with the host's registry. This works only if the host and plugin share the same `PluginRegistry` instance — either via `RTLD_GLOBAL`, via a shared library both link against, or via a raw function pointer to the host's registry passed to the plugin at init time.

### Runtime Discovery via Directory Scan

```cpp
#include <filesystem>

std::vector<void*> loadPluginsFrom(const std::filesystem::path& dir) {
    std::vector<void*> handles;
    for (const auto& entry : std::filesystem::directory_iterator(dir)) {
        if (entry.path().extension() != ".so") continue;
        void* h = dlopen(entry.path().c_str(), RTLD_NOW | RTLD_LOCAL);
        if (!h) {
            fprintf(stderr, "Failed to load %s: %s\n",
                    entry.path().c_str(), dlerror());
            continue;
        }
        // Probe for the factory symbol
        auto* probe = dlsym(h, "plugin_metadata");
        if (!probe) { dlclose(h); continue; }
        handles.push_back(h);
    }
    return handles;
}
```

The `plugin_metadata` probe checks that the shared object is a valid plugin before loading it — a lightweight gate against loading unrelated shared libraries.

### Manifest-Based Discovery

A JSON manifest file lists valid plugins by path, name, and version:

```json
{
  "plugins": [
    { "name": "reverb", "path": "plugins/libaudio_reverb.so.2", "version": 2 },
    { "name": "eq",     "path": "plugins/libaudio_eq.so.1",     "version": 1 }
  ]
}
```

The host reads the manifest, validates paths against an allowlist, and loads only listed plugins. This prevents arbitrary code execution via filesystem injection.

---

## 222.5 Windows and macOS Equivalents

### Windows: LoadLibrary/GetProcAddress

```c
#include <windows.h>

HMODULE handle = LoadLibraryW(L"C:\\plugins\\myplugin.dll");
if (!handle) {
    DWORD err = GetLastError();
    // err = 126 → module not found, err = 193 → not a valid Win32 app
    return;
}

typedef int (*PluginInitFn)(void);
PluginInitFn init = (PluginInitFn)GetProcAddress(handle, "plugin_init");
if (!init) {
    DWORD err = GetLastError();  // err = 127 → procedure not found
    FreeLibrary(handle);
    return;
}

init();
FreeLibrary(handle);
```

Key Windows-specific behaviors:
- `LoadLibraryW` takes a wide-character path; use `LoadLibraryA` for narrow (avoid in new code)
- DLL search order: application directory → `SetDllDirectoryW` paths → system directories → `PATH`
- **DLL Hell**: Windows has no soname versioning; version management is via directory isolation or SxS (side-by-side assemblies)
- **Delay-loaded DLLs**: `/DELAYLOAD:plugin.dll` defers loading to first use; can be used with `__delayLoadHelper2` customization
- `FreeLibrary` decrements the reference count; the DLL is unloaded when it reaches zero

### Windows vs POSIX Comparison

| Feature | POSIX | Windows |
|---------|-------|---------|
| Load | `dlopen` | `LoadLibraryW` |
| Symbol lookup | `dlsym` | `GetProcAddress` |
| Unload | `dlclose` | `FreeLibrary` |
| Error | `dlerror()` | `GetLastError()` |
| Lazy binding | `RTLD_LAZY` | Delay-load import |
| Global symbols | `RTLD_GLOBAL` | No direct equivalent |
| Search path | `LD_LIBRARY_PATH` | `PATH`, `SetDllDirectoryW` |
| ABI versioning | soname + symversioning | Directory isolation / manifests |

### macOS: dlopen with Gatekeeper Constraints

macOS uses the POSIX `dlopen`/`dlsym`/`dlclose` API, but with additional restrictions:

- **Code signing**: on Apple Silicon (arm64), all `dlopen`'d libraries must be signed. Unsigned libraries are rejected at load time on hardened-runtime processes.
- **Gatekeeper**: libraries from outside the app bundle may be quarantined; `xattr -d com.apple.quarantine plugin.dylib` removes quarantine for development.
- **dyld_shared_cache**: system frameworks are pre-linked into a shared cache. You cannot `dlopen` a private framework not in the shared cache on arm64 iOS/macOS.
- **Library Validation (Hardened Runtime)**: `com.apple.security.cs.allow-dyld-environment-variables` entitlement is required to use `DYLD_INSERT_LIBRARIES` on hardened processes.

```bash
# Check if a library is in the dyld shared cache
dyld_info -libraries /path/to/binary

# Sign a plugin for local development
codesign --force --sign - plugin.dylib
```

macOS section name for the shared library identifier: `LC_ID_DYLIB` load command carries the install name. Best practice: set the install name to the expected deployment path:

```bash
clang++ -shared -install_name @rpath/libplugin.dylib \
    -o libplugin.dylib plugin_impl.cpp
```

---

## 222.6 llvm::sys::DynamicLibrary

LLVM provides a cross-platform abstraction over `dlopen`/`LoadLibrary` in `llvm/include/llvm/Support/DynamicLibrary.h`:

```cpp
#include "llvm/Support/DynamicLibrary.h"

// Load a library permanently (added to global search table)
std::string errMsg;
if (llvm::sys::DynamicLibrary::LoadLibraryPermanently(
        "/path/to/libplugin.so", &errMsg)) {
    llvm::errs() << "Failed to load: " << errMsg << "\n";
    return;
}

// Look up a symbol in any permanently loaded library
void* sym = llvm::sys::DynamicLibrary::SearchForAddressOfSymbol("plugin_init");
if (sym) {
    auto* fn = reinterpret_cast<int(*)()>(sym);
    fn();
}
```

### getPermanentLibrary

For handle-based access (needed to close the library later):

```cpp
// Returns a DynamicLibrary handle; the library is loaded permanently
llvm::sys::DynamicLibrary lib =
    llvm::sys::DynamicLibrary::getPermanentLibrary(
        "/path/to/libplugin.so", &errMsg);

// Look up a symbol in this specific library
void* sym = lib.getAddressOfSymbol("plugin_init");
```

### DynamicLibrarySearchGenerator

ORC integrates with `llvm::sys::DynamicLibrary` via `DynamicLibrarySearchGenerator` to expose process symbols to JIT'd code:

```cpp
#include "llvm/ExecutionEngine/Orc/DynamicLibrarySearchGenerator.h"

// Add all symbols from the running process (including dlopen'd libraries)
// to a JITDylib, so JIT'd code can call them
auto& JD = LLJIT->getMainJITDylib();
auto DLSG = llvm::orc::DynamicLibrarySearchGenerator::GetForCurrentProcess(
    LLJIT->getDataLayout().getGlobalPrefix());
if (!DLSG) { handleError(DLSG.takeError()); return; }
JD.addGenerator(std::move(*DLSG));
```

Under the hood, `DynamicLibrarySearchGenerator` calls `llvm::sys::DynamicLibrary::SearchForAddressOfSymbol` to resolve any symbol that is not found in the JITDylib. This allows JIT'd code to call functions in plugins that were loaded via `dlopen`.

### Cross-Platform Notes

`llvm::sys::DynamicLibrary` uses:
- `dlopen`/`dlsym` on Linux, macOS, FreeBSD
- `LoadLibrary`/`GetProcAddress` on Windows
- A custom implementation on platforms without either (e.g., WASM, bare metal — not supported)

It does not expose `RTLD_LOCAL` vs `RTLD_GLOBAL` control; `LoadLibraryPermanently` is equivalent to `RTLD_GLOBAL`. For `RTLD_LOCAL`-equivalent behavior, use `dlopen` directly.

---

## 222.7 Real-World Plugin Architectures

### GCC Plugin API

GCC's plugin API (`gcc/plugin.h`) was the first mainstream compiler plugin interface. A GCC plugin is a shared library with a well-known entry point:

```c
// myplugin.c
#include "gcc-plugin.h"
#include "plugin-version.h"

int plugin_is_GPL_compatible;  // REQUIRED: absence = load failure

struct plugin_info info = { "1.0", "My optimization plugin" };

int plugin_init(struct plugin_name_args* args,
                struct plugin_gcc_version* ver) {
    if (!plugin_default_version_check(ver, &gcc_version)) {
        fprintf(stderr, "Version mismatch\n");
        return 1;
    }
    register_callback(args->base_name, PLUGIN_PASS_MANAGER_SETUP,
                      &my_pass_setup_callback, NULL);
    return 0;
}
```

GCC plugins use `RTLD_NOW | RTLD_GLOBAL` because GCC's internal symbols need to be accessible to the plugin. This is the pattern to avoid in new designs — `RTLD_LOCAL` is safer.

### LLVM's Own Backend Plugins

LLVM supports dynamically loading backend targets and pass plugins at runtime:

```bash
# Load a pass plugin (LLVM 14+)
opt -load-pass-plugin=./MyPassPlugin.so --my-pass input.ll

# Load a target plugin (experimental)
llc --load=./MyTarget.so -march=mytarget input.ll
```

The plugin entry point:

```cpp
// MyPassPlugin.cpp
#include "llvm/Passes/PassPlugin.h"

llvm::PassPluginLibraryInfo getMyPassPluginInfo() {
    return {LLVM_PLUGIN_API_VERSION, "MyPass", "v0.1",
            [](llvm::PassBuilder& PB) {
                PB.registerPipelineParsingCallback(
                    [](llvm::StringRef Name, llvm::ModulePassManager& MPM,
                       llvm::ArrayRef<llvm::PassBuilder::PipelineElement>) {
                        if (Name == "my-pass") {
                            MPM.addPass(MyPass());
                            return true;
                        }
                        return false;
                    });
            }};
}

// The well-known entry point; loaded via dlsym("llvmGetPassPluginInfo")
extern "C" LLVM_ATTRIBUTE_WEAK llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return getMyPassPluginInfo();
}
```

`LLVM_PLUGIN_API_VERSION` is defined in `llvm/include/llvm/Passes/PassPlugin.h` and is checked at load time to ensure ABI compatibility between the plugin and the `opt` binary.

### VST3 Audio Plugin Interface

VST3 (Steinberg) uses a COM-style interface hierarchy:

```cpp
// Simplified VST3 interface (actual SDK is more complex)
namespace Steinberg {
struct IPluginBase : FUnknown {
    virtual tresult PLUGIN_API initialize(FUnknown* context) = 0;
    virtual tresult PLUGIN_API terminate() = 0;
};
struct IAudioProcessor : FUnknown {
    virtual tresult PLUGIN_API setupProcessing(ProcessSetup& setup) = 0;
    virtual tresult PLUGIN_API process(ProcessData& data) = 0;
};
}

// Factory function: well-known C linkage entry point
extern "C" SMTG_EXPORT_SYMBOL
IPluginFactory* PLUGIN_API GetPluginFactory();
```

VST3 uses GUIDs (`FUID`) for interface versioning — a plugin can implement multiple interface versions simultaneously, and the host queries (`queryInterface`) for the highest version it supports. This is the most robust approach to C++ ABI evolution: no binary change breaks old hosts; new hosts get new features.

### LV2 Audio Plugins

LV2 (Linux Audio Developer's Simple Plugin API) takes the opposite approach: purely C, no virtual dispatch, URI-based feature negotiation:

```c
// lv2.h — the entire core interface
typedef struct LV2_Descriptor {
    const char* URI;
    void*       (*instantiate)(const LV2_Descriptor*, double sample_rate,
                               const char* bundle_path,
                               const LV2_Feature* const* features);
    void        (*connect_port)(LV2_Handle instance, uint32_t port, void* data);
    void        (*run)(LV2_Handle instance, uint32_t n_samples);
    void        (*cleanup)(LV2_Handle instance);
    const void* (*extension_data)(const char* uri);
} LV2_Descriptor;

// Well-known entry point:
const LV2_Descriptor* lv2_descriptor(uint32_t index);
```

LV2 is maximally ABI-stable: the `LV2_Descriptor` struct has never changed since publication. Feature negotiation via URIs allows the interface to evolve without binary changes. This is the reference design for stable, extensible C plugin interfaces.

### Game Mod Systems

Quake/id Software-style mods: the game exports a `gameAPI_t` struct of function pointers; the mod DLL receives it at load time and returns its own function pointer table:

```c
// engine exports:
typedef struct {
    void (*print)(const char* msg);
    void (*spawn_entity)(vec3_t origin, int type);
    // ... 100s of functions
} engineAPI_t;

// mod DLL entry point:
modAPI_t* EXPORT Mod_Init(engineAPI_t* engine);
```

This pattern gives the mod access to all engine functionality without knowing the engine's internal symbol names. It is maximally portable (no dynamic linker symbol resolution needed after `Mod_Init`) and ABI-stable as long as the struct is append-only.

---

## 222.8 Security Model and Mitigations

### dlopen as Arbitrary Code Execution

`dlopen("/path/to/plugin.so", RTLD_NOW)` executes all `__attribute__((constructor))` functions in the loaded library before returning. If an attacker controls the path argument, they have arbitrary native code execution in the host process with full privileges.

This is not a bug — it is the intended behavior. The security model for dynamic loading is: **only load libraries from trusted sources**.

### Attack Vectors

1. **Path traversal**: `dlopen(user_input, ...)` allows loading arbitrary `.so` files
2. **Search path hijacking**: if `LD_LIBRARY_PATH` or `rpath` points to a writable directory, an attacker who can write to that directory can inject a malicious library
3. **Plugin directory injection**: if the plugin discovery scan (`std::filesystem::directory_iterator`) runs over a writable directory, attacker-placed `.so` files will be loaded
4. **Network-delivered paths**: accepting a plugin path from a network connection and passing it to `dlopen` is a remote code execution vulnerability

### Mitigations

```cpp
// Allowlist: only load plugins from a trusted path
bool isAllowedPlugin(const std::filesystem::path& p) {
    auto canonical = std::filesystem::canonical(p);  // resolve symlinks
    auto allowlist = std::filesystem::canonical("/usr/lib/myapp/plugins");
    // Check that canonical path starts with the allowlist prefix
    auto [a, b] = std::mismatch(allowlist.begin(), allowlist.end(),
                                canonical.begin());
    return a == allowlist.end();
}
```

Platform-specific hardening:
- **Linux IMA/fs-verity**: kernel-level file integrity measurement; blocks execution of unsigned code
- **macOS mandatory code signing**: Hardened Runtime (`com.apple.security.cs.hardened-runtime`) rejects loading unsigned dylibs
- **SELinux `EXECMEM` denial**: prevents `mprotect` calls that change pages from non-executable to executable — blocks dlopen on systems with this policy
- **seccomp filter**: a process can block `openat` system calls on untrusted paths using a seccomp-BPF filter
- **Privilege-separated plugin runner**: run the plugin in a separate process (fork + execve into a sandboxed environment); communicate via IPC (shared memory, pipe, or Unix socket). The plugin's sandbox has limited capabilities; compromise of the plugin does not compromise the host

### Cryptographic Signature Verification

For the highest assurance, verify a cryptographic signature over the plugin file before loading:

```cpp
bool verifyPluginSignature(const std::filesystem::path& p) {
    // Read the plugin file
    auto contents = readFile(p);
    // Read the accompanying .sig file
    auto sig = readFile(p.string() + ".sig");
    // Verify Ed25519 signature against pinned public key
    return crypto_sign_verify_detached(
        sig.data(), contents.data(), contents.size(), pinnedPublicKey) == 0;
}

if (!verifyPluginSignature(pluginPath)) {
    fprintf(stderr, "Plugin signature verification failed\n");
    return;
}
dlopen(pluginPath.c_str(), RTLD_NOW | RTLD_LOCAL);
```

This approach — load only cryptographically signed plugins — is the security model used by Apple's App Store (for app extensions) and by browser extension systems.

---

## 222.9 Interaction with ORC JIT and Self-Modification

Dynamic loading via `dlopen` and ORC JIT are complementary mechanisms that can be combined:

| Property | dlopen | ORC JIT |
|----------|--------|---------|
| Granularity | Shared library (.so) | Module (arbitrary size) |
| Compilation time | AOT (pre-compiled) | JIT (at load time) |
| ABI | Stable, versioned | Flexible, runtime-generated |
| Symbol hot-swap | No (must reload library) | Yes (RedirectableSymbolManager) |
| Source required | No | Optional (bitcode or source) |
| W^X handling | OS-managed | JITLink handles explicitly |

### Combined Architecture: Stable ABI + Mutable Implementation

The most powerful combination:

1. **Host** exports a stable C-linkage plugin interface (§222.3)
2. **Plugin** loaded via `dlopen` contains an ORC `ExecutionSession`
3. The plugin stub (the `dlopen`'d code) is ABI-stable and unchanging
4. Behind the stub, the ORC session compiles and hot-swaps implementations dynamically
5. The host calls the same `extern "C"` entry points; the plugin stub delegates to the ORC-compiled implementation

```cpp
// Inside the plugin (plugin_stub.cpp):
static llvm::orc::LLJIT* gJIT = nullptr;
static llvm::orc::RedirectableSymbolManager* gRSM = nullptr;

extern "C" int plugin_process(PluginHandle h, const float* in, float* out, int n) {
    // Look up the current JIT-compiled implementation
    using ProcessFn = int(*)(void*, const float*, float*, int);
    auto sym = llvm::cantFail(gJIT->lookup("process_impl"));
    auto fn = sym.toPtr<ProcessFn>();
    return fn(h, in, out, n);
}
```

The `dlopen`/`dlsym` ABI never changes. The ORC layer beneath it hot-swaps `process_impl` using `redirect()` without the host knowing. The host only ever calls `plugin_process` via a stable C-linkage function pointer.

### DynamicLibrarySearchGenerator in the Combined Model

When the plugin's ORC session needs to call functions in the host process (or in other plugins), `DynamicLibrarySearchGenerator` resolves them via the process-wide symbol table:

```cpp
// Inside the plugin's ORC setup:
auto& JD = gJIT->getMainJITDylib();
auto DLSG = llvm::orc::DynamicLibrarySearchGenerator::GetForCurrentProcess(
    gJIT->getDataLayout().getGlobalPrefix());
JD.addGenerator(std::move(*DLSG));
// JIT'd code can now call host functions by name
```

This creates a two-level symbol resolution: the ORC JITDylib for JIT'd symbols, and the process symbol table (via `DynamicLibrarySearchGenerator`) for everything else. The combined model allows plugin implementations to call any host API without any compile-time linking.

---

## Chapter Summary

- `dlopen`/`dlsym`/`dlclose` is the POSIX dynamic loading API; `RTLD_NOW` (fail-fast) and `RTLD_LOCAL` (hermetic) are the safe production defaults; `RTLD_GLOBAL` exposes plugin symbols globally and should be used carefully
- The runtime linker searches `DT_RPATH`, `LD_LIBRARY_PATH`, `DT_RUNPATH`, `/etc/ld.so.cache`, and system paths in order; production plugin loaders should use absolute paths or `$ORIGIN`-relative `RUNPATH`
- The ABI contract between host and plugin must survive different compiler versions; `extern "C"` C linkage is the lingua franca — no name mangling, no C++ ABI dependence
- Soname versioning (`libplugin.so.1.2.3` → SONAME `libplugin.so.1`) manages the major/minor/patch ABI versioning contract; symbol versioning scripts provide function-level compatibility control
- Type-safe C++ plugin interfaces use the abstract factory pattern: an `extern "C"` factory returns an abstract base pointer; `std::unique_ptr<IPlugin, DestroyFn>` manages lifetime; `std::string`, exceptions, and RTTI cannot safely cross plugin boundaries
- Plugin discovery uses `__attribute__((constructor))` registration, `std::filesystem` directory scans with a `plugin_metadata` probe, or manifest files; manifest-based discovery is the most secure
- Windows uses `LoadLibraryW`/`GetProcAddress`/`FreeLibrary`; macOS uses POSIX `dlopen` with code-signing requirements on arm64; `llvm::sys::DynamicLibrary` provides a cross-platform abstraction
- Real-world reference architectures: LLVM pass plugins (`llvmGetPassPluginInfo`), VST3 (COM-style GUID interface versioning), LV2 (pure C, URI-based feature negotiation), game mods (function-pointer struct exchange at init)
- `dlopen` on an attacker-controlled path is arbitrary code execution; mitigations include path allowlisting (canonical path prefix check), cryptographic signature verification, and privilege-separated sandbox runners
- ORC JIT and `dlopen` are complementary: the plugin stub exposes a stable `extern "C"` ABI; behind it, an ORC `ExecutionSession` hot-swaps implementations via `RedirectableSymbolManager::redirect()` without the host observing any ABI change; `DynamicLibrarySearchGenerator` resolves host symbols for JIT'd code

---

@copyright jreuben11
