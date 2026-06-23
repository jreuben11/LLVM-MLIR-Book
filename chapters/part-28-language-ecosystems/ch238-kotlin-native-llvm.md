# Chapter 238 — Kotlin/Native: Compiling Kotlin to Native Code via LLVM

*Part XXVIII — Language Ecosystems and Alternative Front-Ends*

Kotlin/Native compiles Kotlin source into native binaries — executables, shared libraries, static libraries, and Apple frameworks — without a JVM. It uses LLVM as its back-end, making it the primary route for Kotlin programs that must run on iOS, embedded Linux, or Windows without a Java runtime. The compiler chain runs K2 FIR (JetBrains' redesigned front-end IR) through several Kotlin-specific IR transformations, then emits LLVM IR via a custom LLVM fork, applies ThinLTO, and links with LLD or the platform linker. This chapter covers the full pipeline from `KonanBackend` to final binary, including the memory manager transition, C interop, Objective-C bridging, and multi-target output modes.

---

## Table of Contents

- [238.1 KonanBackend, K2 FIR, and KonanTarget](#2381-konanbackend-k2-fir-and-konantarget)
  - [K2 compiler pipeline](#k2-compiler-pipeline)
  - [KonanTarget enum](#konantarget-enum)
- [238.2 IR Lowerings, Interop, and LTO](#2382-ir-lowerings-interop-and-lto)
  - [Kotlin IR lowering passes](#kotlin-ir-lowering-passes)
  - [ModuleBitcodeOptimization and LTOBitcodeOptimization](#modulebitcodeoptimization-and-ltobitcodeoptimization)
- [238.3 CodeGenerator.kt and the Custom LLVM Fork](#2383-codegeneratorkt-and-the-custom-llvm-fork)
  - [Function attributes for K/N calling convention](#function-attributes-for-kn-calling-convention)
- [238.4 Memory Manager: ARC/Freeze to Concurrent GC](#2384-memory-manager-arcfreeze-to-concurrent-gc)
  - [Pre-1.7.20: Strict ARC with thread isolation](#pre-1720-strict-arc-with-thread-isolation)
  - [1.7.20+: Concurrent mark-and-sweep GC](#1720-concurrent-mark-and-sweep-gc)
- [238.5 GC Safepoints and Stack Maps](#2385-gc-safepoints-and-stack-maps)
- [238.6 C Interop with cinterop Tool and .def Files](#2386-c-interop-with-cinterop-tool-and-def-files)
  - [.def files](#def-files)
  - [Using the interop library](#using-the-interop-library)
  - [kotlinx.cinterop type hierarchy](#kotlinxcinterop-type-hierarchy)
- [238.7 Objective-C and Swift Interop](#2387-objective-c-and-swift-interop)
  - [Calling ObjC from Kotlin](#calling-objc-from-kotlin)
  - [@ObjCName and @ExportObjCClass](#objcname-and-exportobjcclass)
  - [Swift interop via generated headers](#swift-interop-via-generated-headers)
- [238.8 KonanTarget Output Modes and XCFramework](#2388-konantarget-output-modes-and-xcframework)
  - [Output modes via -produce flag](#output-modes-via-produce-flag)
  - [XCFramework](#xcframework)
  - [@CName for C-visible exports](#cname-for-c-visible-exports)
  - [Debugging LLVM IR generation](#debugging-llvm-ir-generation)
- [Summary](#summary)

---

## 238.1 KonanBackend, K2 FIR, and KonanTarget

### K2 compiler pipeline

Since Kotlin 2.0, the Kotlin/Native compiler uses the **K2 front end**. The pipeline stages are:

```
Kotlin source
    ↓  K2 FIR (Frontend IR) — type checking, resolution, diagnostics
    ↓  Fir2Ir — FIR → IrElement (Kotlin IR, "KIR")
    ↓  KonanFirExtensions — K/N-specific FIR checkers
    ↓  IrGenerationExtension.generate() — plugin injection point
    ↓  Kotlin IR lowering passes (40+ passes)
    ↓  BitcodePostProcessing → LLVM Module
    ↓  LTO (ModuleBitcodeOptimization / LTOBitcodeOptimization)
    ↓  Link (LLD or platform linker)
    ↓  Native binary
```

**FIR (Frontend IR)** is JetBrains' tree-based front-end representation that replaced the older PSI-based front end. It carries full type information including nullability, variance, and platform types. `Fir2Ir` converts FIR trees to `IrElement` nodes, which are the entry point for all backend lowering.

The `KonanBackend` class (`backend/konan/src/org/jetbrains/kotlin/backend/konan/KonanBackend.kt`) owns the full compilation context. It instantiates the `Context` object that carries the `KonanConfig`, `IrModule`, and the `LlvmDeclarations` table mapping Kotlin IR elements to LLVM `Value*` pointers.

### KonanTarget enum

`KonanTarget` enumerates all supported platforms:

```kotlin
// backend/konan/src/org/jetbrains/kotlin/backend/konan/KonanTarget.kt
sealed class KonanTarget(
    val name: String,
    val family: Family,
    val architecture: Architecture,
    val targetTriple: TargetTriple,
) {
    object IOS_ARM64        : KonanTarget("ios_arm64",       Family.IOS,     Architecture.ARM64, ...)
    object IOS_X64          : KonanTarget("ios_x64",         Family.IOS,     Architecture.X64,   ...)
    object IOS_SIMULATOR_ARM64 : KonanTarget(...)
    object MACOS_X64        : KonanTarget("macos_x64",       Family.OSX,     Architecture.X64,   ...)
    object MACOS_ARM64      : KonanTarget("macos_arm64",     Family.OSX,     Architecture.ARM64, ...)
    object LINUX_X64        : KonanTarget("linux_x64",       Family.LINUX,   Architecture.X64,   ...)
    object LINUX_ARM64      : KonanTarget("linux_arm64",     Family.LINUX,   Architecture.ARM64, ...)
    object ANDROID_ARM64    : KonanTarget("android_arm64",   Family.ANDROID, Architecture.ARM64, ...)
    object ANDROID_ARM32    : KonanTarget("android_arm32",   Family.ANDROID, Architecture.ARM32, ...)
    object MINGW_X64        : KonanTarget("mingw_x64",       Family.MINGW,   Architecture.X64,   ...)
    object WATCHOS_ARM64    : KonanTarget(...)
    object TVOS_ARM64       : KonanTarget(...)
    object WASM32           : KonanTarget("wasm32",          Family.WASM,    Architecture.WASM32, ...)
    // ... ~20 total
}
```

The `Family` enum drives platform-specific default behaviors: Apple families get Objective-C interop enabled; ANDROID family gets the NDK sysroot; MINGW gets PE/COFF output.

---

## 238.2 IR Lowerings, Interop, and LTO

### Kotlin IR lowering passes

Between `Fir2Ir` output and LLVM emission, the compiler runs approximately 40 lowering passes registered in `KonanLoweringPhases`. Key passes include:

| Pass | Effect |
|------|--------|
| `InteropLowering` | Replaces `kotlinx.cinterop` calls with direct LLVM intrinsics |
| `BridgesBuilding` | Inserts bridge functions for virtual dispatch across inline class boundaries |
| `CoroutinesLowering` | Converts `suspend` functions to state machines |
| `EnumsLowering` | Converts Kotlin enums to sealed integer types |
| `InlineClassLowering` | Unwraps value class wrappers |
| `ObjectsLowering` | Materialises Kotlin `object` singletons with thread-safe initialization |
| `AutoboxingLowering` | Box/unbox primitive types for generic containers |
| `PropertyDelegationLowering` | Inlines delegated property accessors |
| `GCRootHolderLowering` | Inserts GC root registrations for heap references in frames |

`InteropLowering` is particularly important: it handles `CPointer<T>`, `COpaquePointer`, `CFunction<T>`, and `interpretCPointer` calls, replacing them with typed LLVM IR pointer arithmetic.

### ModuleBitcodeOptimization and LTOBitcodeOptimization

After IR lowering, LLVM IR is serialised to bitcode and optimised:

```kotlin
// backend/konan/src/org/jetbrains/kotlin/backend/konan/BitcodePostProcessing.kt
fun optimizeModule(context: Context, llvmModule: LLVMModuleRef) {
    ModuleBitcodeOptimization(context, llvmModule).run()
}
```

`ModuleBitcodeOptimization` runs the per-module pipeline (inlining, SROA, GVN, loop vectorisation at the configured opt level). `LTOBitcodeOptimization` runs ThinLTO across all modules when LTO is enabled (the default for release builds).

Kotlin/Native's custom LLVM fork (`jetbrains/llvm-project`) carries patches for:
- GC statepoint intrinsics tailored to the concurrent GC (§238.5)
- `objcretainblock` handling for Objective-C block semantics
- ARM64 return-address signing for Apple platforms

The flag `-Xsave-llvm-ir-after=<pass>` dumps LLVM IR after a named pass, which is essential for debugging lowering correctness or investigating unexpected codegen.

---

## 238.3 CodeGenerator.kt and the Custom LLVM Fork

`CodeGenerator.kt` (`backend/konan/src/org/jetbrains/kotlin/backend/konan/llvm/CodeGenerator.kt`) is the central IR emitter. It wraps LLVM C API calls in Kotlin helper methods:

```kotlin
class CodeGenerator(val context: Context) {
    val llvmContext: LLVMContextRef = LLVMGetGlobalContext()
    val llvmModule: LLVMModuleRef   = LLVMModuleCreateWithNameInContext("main", llvmContext)
    val llvmBuilder: LLVMBuilderRef = LLVMCreateBuilderInContext(llvmContext)

    fun functionType(returnType: LLVMTypeRef, vararg paramTypes: LLVMTypeRef): LLVMTypeRef =
        LLVMFunctionType(returnType, paramTypes.toList().toTypedArray(), paramTypes.size, 0)

    fun intPtrType(): LLVMTypeRef = LLVMIntPtrTypeInContext(llvmContext, llvmTargetData)

    fun emitLoad(ptr: LLVMValueRef, name: String = "", isVolatile: Boolean = false): LLVMValueRef =
        LLVMBuildLoad2(llvmBuilder, getElementType(ptr), ptr, name)
            .also { if (isVolatile) LLVMSetVolatile(it, 1) }
}
```

The custom LLVM fork is pinned to a specific LLVM version (currently LLVM 16 for K/N 2.x; migration to 17/18 is tracked in YouTrack). The fork lives at `github.com/JetBrains/llvm-project` and is consumed as a prebuilt binary in the Kotlin/Native distribution.

### Function attributes for K/N calling convention

Kotlin/Native uses the standard C calling convention for all generated functions. However, several attributes are applied universally:

```llvm
; Typical Kotlin/Native function attributes
define void @kotlin.mypackage.MyClass.myMethod(%struct.ObjHeader** %thiz) #0 {
  ; ...
}

attributes #0 = {
  nounwind                    ; Kotlin exception handling is Objective-C / SEH, not C++ EH
  "frame-pointer"="all"       ; Required for GC stack scanning
  "no-trapping-math"="true"
}
```

`"frame-pointer"="all"` is mandatory: the GC stack scanner traverses frame chains to find all live pointers, and omitting frame pointers would break this traversal.

---

## 238.4 Memory Manager: ARC/Freeze to Concurrent GC

### Pre-1.7.20: Strict ARC with thread isolation

Before Kotlin/Native 1.7.20, the memory model required that heap objects belong to exactly one thread or be frozen. Frozen objects were immutable and could be shared. The collector was a thread-local reference-counting ARC scheme:

```kotlin
// Old model — runtime error in pre-1.7.20
val shared = SomeObject()
shared.freeze()  // makes immutable, allows sharing
val job = GlobalScope.launch {
    println(shared.value)  // OK — shared is frozen
}
```

Unfrozen objects passed to another coroutine would throw `IncorrectDereferenceException` at runtime. This made concurrent code difficult and was the primary source of K/N user complaints.

### 1.7.20+: Concurrent mark-and-sweep GC

Kotlin/Native 1.7.20 introduced a new memory manager (`-memory-model=experimental`, promoted to default in 2.0) based on a **concurrent mark-and-sweep** GC:

- Objects are allocated from a thread-local pool first (fast path); overflow goes to the global heap.
- The GC runs concurrently on a background thread, tracing from root sets while mutator threads continue.
- Stop-the-world pauses are limited to root set enumeration (a few microseconds for typical programs).
- The `freeze()` API still works but is now a no-op — all objects are shareable by default.

The concurrent GC is implemented in `kotlin-native/runtime/src/gc/cms/` (CMS = Concurrent Mark-and-Sweep). Key files:

- `ConcurrentMarkAndSweep.cpp` — GC thread loop, mark queue, sweep
- `GCImpl.hpp` — public GC API used by the allocator and write barriers
- `Barriers.cpp` — write barrier implementation (Dijkstra-style: flag on pointer write)

---

## 238.5 GC Safepoints and Stack Maps

The concurrent GC must stop mutator threads at **safepoints** — program points where all thread state (registers, stack) can be scanned consistently. Kotlin/Native's safepoint mechanism uses LLVM's `gc.statepoint` intrinsic family:

```llvm
; Statepoint call — LLVM can insert safepoint poll here
%tok = call token @llvm.experimental.gc.statepoint.p0f_i32f(
    i64 0, i32 0,          ; id, # patch bytes
    i32 ()* @some_function,
    i32 0, i32 0,           ; # call args, # transition args
    i32 0,                  ; # deopt args
    i32* %gcref1, ...)      ; live GC values

; Reload GC references after statepoint (may have been relocated)
%new_ref = call i32* @llvm.experimental.gc.relocate.p0i32(
    token %tok, i32 0, i32 0)
```

The `CoroPoison` pass in Kotlin/Native's LLVM fork patches the statepoint sequence to match K/N's exact GC root layout. After `CoroSplit`-equivalent processing, LLVM's `StackMaps` section in the output object file records the exact register and stack-slot locations of live GC references at each statepoint.

The runtime's `StackMapParser.cpp` reads the `__llvm_stackmaps` section at startup to build the stack-map table used by the concurrent GC's stop phase.

---

## 238.6 C Interop with cinterop Tool and .def Files

Kotlin/Native's C interop uses a separate `cinterop` tool that wraps libclang to parse C headers and generate Kotlin binding stubs.

### .def files

A `.def` file describes what to import:

```ini
# curl.def
headers = curl/curl.h
headerFilter = curl/**
linkerOpts.linux = -lcurl
linkerOpts.macos = -lcurl
compilerOpts = -I/usr/include

# Inline C helpers (for macros that aren't parseable as Kotlin)
---
#include <curl/curl.h>
static size_t write_cb(void *ptr, size_t sz, size_t nmemb, void *data) {
    return ((KotlinWriteCallback)data)(ptr, sz * nmemb);
}
```

Running `cinterop -def curl.def -target linux_x64 -o curl.klib` produces a `.klib` (Kotlin Library archive) containing:
- Kotlin stubs with types like `CPointer<CURLType>`, `CFunction<(COpaquePointer?) -> Long>`
- The pre-compiled bitcode for the inline C helpers

### Using the interop library

```kotlin
import libcurl.*

fun main() {
    val curl: CURLVar = curl_easy_init() ?: return
    curl_easy_setopt(curl, CURLOPT_URL, "https://example.com")
    curl_easy_perform(curl)
    curl_easy_cleanup(curl)
}
```

### kotlinx.cinterop type hierarchy

The `kotlinx.cinterop` package provides the type system for interop:

| Kotlin type | C equivalent |
|-------------|--------------|
| `CPointer<T>` | `T*` |
| `COpaquePointer` | `void*` |
| `CPointerVar<T>` | `T**` (addressof a pointer) |
| `ByteVar` | `char` / `int8_t` |
| `IntVar` | `int` |
| `CStructVar` (abstract) | `struct { ... }` |
| `CFunction<(A, B) -> R>` | function pointer `R (*)(A, B)` |

Memory for C types is allocated with `memScoped { ... }` (arena-allocated, freed at scope exit) or `nativeHeap.alloc<T>()` (manually managed). TinyGo's `CPointer<T>` maps directly to LLVM `ptr addrspace(0)` during `InteropLowering`.

---

## 238.7 Objective-C and Swift Interop

On Apple targets, Kotlin/Native has first-class Objective-C (ObjC) interoperability. The `cinterop` tool uses `libclang` to parse Objective-C headers and generates idiomatic Kotlin bindings.

### Calling ObjC from Kotlin

```kotlin
import platform.UIKit.*
import platform.Foundation.*

class MyViewController : UIViewController() {
    override fun viewDidLoad() {
        super.viewDidLoad()
        val label = UILabel(frame = CGRectMake(0.0, 0.0, 200.0, 50.0))
        label.text = "Hello from Kotlin/Native"
        view.addSubview(label)
    }
}
```

`UIViewController` and `UILabel` are generated ObjC stubs. The `cinterop`-generated code emits `objc_msgSend` calls through LLVM IR:

```llvm
; [label setText:@"Hello"]
%sel = call i8* @sel_getUid(i8* getelementptr ([5 x i8], [5 x i8]* @.str.setText_, i32 0, i32 0))
call void bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to void (i8*, i8*, i8*)*)(
    i8* %label_ptr, i8* %sel, i8* %nsstring_ptr)
```

### @ObjCName and @ExportObjCClass

Kotlin classes can be exposed to ObjC/Swift using these annotations:

```kotlin
@ObjCName("KTMyProcessor", exact = true)
class MyProcessor {
    @ObjCName("processData")
    fun process(data: ByteArray): String { ... }
}
```

```kotlin
@ExportObjCClass("KTDelegate")
class AppDelegate : NSApplicationDelegateProtocol {
    override fun applicationDidFinishLaunching(notification: NSNotification) {
        // ...
    }
}
```

The `@ExportObjCClass` annotation causes the compiler to register the class with the ObjC runtime via `objc_allocateClassPair` / `objc_registerClassPair` calls in the module initialiser. Swift code can then use `KTMyProcessor` directly without any bridging header.

### Swift interop via generated headers

When producing a framework, Kotlin/Native generates an Objective-C umbrella header that Swift can import:

```objc
// MyFramework.h (generated)
@interface KTMyProcessor : NSObject
- (NSString *)processData:(NSData *)data;
@end
```

Swift sees this as a native ObjC class and the Kotlin implementation is invoked transparently.

---

## 238.8 KonanTarget Output Modes and XCFramework

### Output modes via -produce flag

| Mode | Flag | Output |
|------|------|--------|
| Executable | `-produce program` | Native binary (`a.out`, `.exe`) |
| Dynamic library | `-produce dynamic` | `.so`, `.dylib`, `.dll` |
| Static library | `-produce static` | `.a`, `.lib` |
| Apple framework | `-produce framework` | `.framework` bundle |
| LLVM bitcode | `-produce bitcode` | `.bc` for external LTO |
| Kotlin library | `-produce library` | `.klib` for Kotlin consumption |

In Gradle (Kotlin Multiplatform):

```kotlin
kotlin {
    iosArm64 {
        binaries {
            framework("MyFramework") {
                baseName = "MyFramework"
                isStatic = true
                freeCompilerArgs += listOf("-Xsave-llvm-ir-after=BitcodeOptimization")
            }
        }
    }
    macosArm64 { binaries { framework("MyFramework") } }
    macosX64  { binaries { framework("MyFramework") } }
}
```

### XCFramework

An XCFramework bundles frameworks for multiple architectures into a single distributable:

```kotlin
// build.gradle.kts
val xcf = XCFramework("MyFramework")
listOf(
    iosArm64(),
    iosSimulatorArm64(),
    iosX64(),
    macosArm64(),
    macosX64()
).forEach {
    it.binaries.framework { xcf.add(this) }
}
```

Running `./gradlew assembleMyFrameworkReleaseXCFramework` invokes `xcodebuild -create-xcframework` to merge the per-target `.framework` directories into a single `MyFramework.xcframework` that Xcode distributes via Swift Package Manager.

### @CName for C-visible exports

To export a Kotlin function with a specific C symbol name (for use from C or other native languages):

```kotlin
import kotlinx.cinterop.ExperimentalForeignApi
import kotlin.native.CName

@OptIn(ExperimentalForeignApi::class)
@CName("mylib_process")
fun process(data: CPointer<ByteVar>, len: Int): Int {
    // ...
}
```

The compiler emits `@mylib_process` as a global LLVM function with the given name, skipping the usual Kotlin name-mangling scheme.

### Debugging LLVM IR generation

The `-Xsave-llvm-ir-after=<pass>` flag dumps the LLVM module at a named pipeline stage. Common useful checkpoints:

```bash
kotlinc-native main.kt -target linux_x64 \
    -Xsave-llvm-ir-after=BitcodeOptimization \
    -Xsave-llvm-ir-after=LTO \
    -o main.kexe
```

This produces `main.bc.BitcodeOptimization.ll` and `main.bc.LTO.ll` in the output directory, allowing inspection with `opt --print-ir` or `llvm-dis`.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **LLVM 18/19 fork migration**: The JetBrains LLVM fork (`github.com/JetBrains/llvm-project`) is tracked in YouTrack KT-65995 for migration from the current LLVM 16 base to LLVM 18/19, unlocking opaque-pointer IR, improved MemorySSA, and new AArch64 MTE/BTI codegen attributes relevant to iOS security hardening.
- **Swift direct interop (KT-67750)**: JetBrains' "Swift Export" experimental feature generates Swift modules directly from Kotlin declarations, bypassing the ObjC umbrella-header indirection; expected to stabilize and exit experimental status in Kotlin 2.2/2.3, eliminating the `@ObjCName` annotation workaround for most Apple-target use cases.
- **Concurrent GC incremental mark phase**: The CMS GC in `kotlin-native/runtime/src/gc/cms/` is gaining incremental marking (splitting the mark phase across multiple GC cycles) to eliminate the remaining stop-the-world pauses that occur on large object graphs; tracked in YouTrack KT-49234.
- **WASM GC target stabilization**: The `wasm32` target is being superseded by `wasmJs` and `wasmWasi` targets leveraging the WASM GC proposal; `KonanTarget.WASM_WASI` is expected to exit alpha and become fully supported alongside removal of the legacy `wasm32` ABI.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Upstream LLVM statepoint graduation**: The `llvm.experimental.gc.statepoint` intrinsic family (central to Kotlin/Native's safepoint mechanism in §238.5) is the subject of ongoing LLVM RFC discussions to graduate it from experimental status; if accepted, JetBrains can drop their statepoint-patching fork commits and track upstream LLVM more closely.
- **K2 IR serialization for separate compilation**: JetBrains is developing `.klib`-level IR serialization that stores Kotlin IR (not just metadata) so that LLVM codegen can be deferred to link time, enabling true per-module lazy compilation and faster incremental builds analogous to Swift's `.swiftmodule` model; expected to require new `.klib` format version.
- **Linux RISC-V 64 (`linux_riscv64`) tier-1 support**: With RISC-V gaining embedded Linux traction, a new `KonanTarget.LINUX_RISCV64` is planned once the Kotlin/Native runtime and GC safepoint scanning are validated on RISC-V's frame unwinding conventions; initial patches exist in feature branches.
- **ARC-to-GC bridge elimination**: Currently, Apple-target builds still carry residual ARC retain/release calls around ObjC object crossings. A planned `ObjCRetainElimination` pass in `CodeGenerator.kt` will prove through alias analysis that Kotlin's GC already covers the lifetime, removing redundant `objc_retain`/`objc_release` pairs and reducing binary size on iOS by an estimated 3–8%.

### 5-Year Horizon (Long-Term, by ~2031)

- **ClangIR (CIR) as shared mid-level IR**: If ClangIR (Chapter 220) matures into a stable interchange layer, Kotlin/Native's `InteropLowering` pass could lower `kotlinx.cinterop` calls through CIR rather than emitting LLVM IR directly, gaining access to Clang's type-layout infrastructure and reducing the bespoke C-ABI logic currently duplicated in the JetBrains fork.
- **Ahead-of-time profile-guided optimization (PGO) via Kotlin instrumentation**: Kotlin/Native currently relies on LLVM's sample-PGO via `llvm-profdata`; long-term plans include a Kotlin-aware PGO that instruments at the Kotlin IR level (before `InteropLowering`) to preserve semantic context across the ~40 lowering passes, enabling profile-guided inlining of virtual dispatch sites that are opaque to LLVM's `pgo-instr-gen`.
- **Formal verification of the concurrent GC write barrier**: As Kotlin/Native targets safety-critical embedded Linux use cases, a Coq or Iris proof of the Dijkstra-style write barrier in `Barriers.cpp` (§238.4) is a long-term research goal analogous to the verified GC work in the Mu micro-virtual-machine project; JetBrains Research has ongoing collaboration with academic groups on this.

---

## Summary

- Kotlin/Native's pipeline is K2 FIR → `Fir2Ir` → 40+ Kotlin IR lowerings → LLVM IR emission → ThinLTO → native binary; `KonanBackend` owns the compilation context.
- `KonanTarget` enumerates ~20 targets across iOS, macOS, watchOS, tvOS, Linux, Android, MinGW, and Wasm families.
- `InteropLowering` handles `kotlinx.cinterop` C pointer types; `ModuleBitcodeOptimization` and `LTOBitcodeOptimization` handle per-module and cross-module LLVM optimization.
- `CodeGenerator.kt` wraps the LLVM C API; the JetBrains LLVM fork adds GC statepoint patches and ObjC retain-block support.
- The memory model migrated from thread-isolated ARC+freeze (pre-1.7.20) to concurrent mark-and-sweep GC (1.7.20+, default in 2.0).
- GC safepoints use `llvm.experimental.gc.statepoint`; `__llvm_stackmaps` sections record live GC root locations per safepoint.
- The `cinterop` tool parses C/ObjC headers via libclang and generates `.klib` stubs; `kotlinx.cinterop` types (`CPointer<T>`, `CStructVar`, etc.) map to LLVM pointer types after `InteropLowering`.
- On Apple targets, `@ObjCName`, `@ExportObjCClass`, and generated ObjC headers give Kotlin classes native Swift visibility; XCFramework bundles multi-target frameworks for distribution.
