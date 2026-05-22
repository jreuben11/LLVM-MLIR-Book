# Chapter 236 — Android NDK: Cross-Compiling C/C++ for Android with LLVM

*Part XXVIII — Language Ecosystems and Engineering Practice*

The Android NDK ships a complete LLVM/Clang toolchain that has been the sole supported native compiler since NDK r18 (2018). Every C/C++/assembly file in an Android native library passes through Clang, LLD, and a Bionic-linked sysroot maintained by Google's Android LLVM team. Understanding the NDK's toolchain layout, target-triple conventions, sysroot model, and Bionic divergences from glibc is essential for writing correct, performant native Android code and for debugging the subtle failures that arise when these details are overlooked.

---

## Table of Contents

- [236.1 NDK Architecture and Toolchain Layout](#2361-ndk-architecture-and-toolchain-layout)
- [236.2 Target Triples and API Levels](#2362-target-triples-and-api-levels)
- [236.3 Sysroot Structure and Stub Libraries](#2363-sysroot-structure-and-stub-libraries)
- [236.4 CMake Integration](#2364-cmake-integration)
  - [Minimum CMakeLists.txt](#minimum-cmakeliststxt)
  - [Gradle configuration](#gradle-configuration)
  - [Key CMake variables](#key-cmake-variables)
- [236.5 Bionic libc Differences](#2365-bionic-libc-differences)
  - [Missing POSIX features](#missing-posix-features)
  - [Register x18 — shadow call stack](#register-x18-shadow-call-stack)
  - [`__attribute__((visibility("default")))` and `-fvisibility=hidden`](#attributevisibilitydefault-and-fvisibilityhidden)
  - [Thread-local storage](#thread-local-storage)
- [236.6 Multi-ABI APK Layout](#2366-multi-abi-apk-layout)
- [236.7 Sanitizers and wrap.sh](#2367-sanitizers-and-wrapsh)
  - [HWAddressSanitizer (HWASan)](#hwaddresssanitizer-hwasan)
  - [wrap.sh mechanism](#wrapsh-mechanism)
  - [AddressSanitizer](#addresssanitizer)
  - [UBSan and CFI](#ubsan-and-cfi)
- [236.8 The Android LLVM Fork and GKI](#2368-the-android-llvm-fork-and-gki)
  - [Android's LLVM fork](#androids-llvm-fork)
  - [Generic Kernel Image (GKI) mandate](#generic-kernel-image-gki-mandate)
  - [Clang-built kernel and LTO](#clang-built-kernel-and-lto)
- [Summary](#summary)

---

## 236.1 NDK Architecture and Toolchain Layout

The NDK is a self-contained directory tree. Since NDK r23 the structure has stabilised:

```
android-ndk-r28/
├── toolchains/
│   └── llvm/
│       └── prebuilt/
│           └── linux-x86_64/
│               ├── bin/
│               │   ├── aarch64-linux-android35-clang
│               │   ├── aarch64-linux-android35-clang++
│               │   ├── armv7a-linux-androideabi21-clang
│               │   ├── x86_64-linux-android21-clang
│               │   ├── i686-linux-android21-clang
│               │   ├── llvm-ar, llvm-nm, llvm-objcopy, llvm-strip
│               │   └── ld.lld
│               ├── sysroot/
│               │   ├── usr/include/
│               │   └── usr/lib/
│               └── lib/clang/22/
├── build/
│   ├── cmake/
│   │   └── android.toolchain.cmake
│   └── ndk-build
├── platforms/         (deprecated, kept for compatibility)
└── sources/
    ├── android/
    └── third_party/
```

Each `<triple><api>-clang` wrapper is a shell script (or symlink on some hosts) that invokes the real `clang` binary with `--target=<triple>` and `--sysroot=<sysroot>` pre-populated. Using the wrapper scripts is strongly recommended over passing these flags manually — they guarantee the correct sysroot path and default library search order.

LLD has been the default linker since NDK r22. It is invoked as `ld.lld` but also responds to the `lld` name. The legacy `bfd` and `gold` linkers are no longer shipped.

The NDK's Clang version tracks slightly behind upstream LLVM mainline but has historically stayed within one major release. NDK r28 ships Clang 22.x built from `android.googlesource.com/toolchain/llvm-project` (see §236.8).

---

## 236.2 Target Triples and API Levels

Android uses four ABI variants, each with a canonical LLVM target triple:

| ABI | Triple | Min Supported API |
|-----|--------|-------------------|
| arm64-v8a | `aarch64-linux-android<api>` | 21 |
| armeabi-v7a | `armv7a-linux-androideabi<api>` | 16 |
| x86_64 | `x86_64-linux-android<api>` | 21 |
| x86 | `i686-linux-android<api>` | 16 |

The API level integer embedded in the triple is the **minimum API level** the binary will run on. It controls:

- Which symbols are available from stub libraries (§236.3)
- Whether `__android_api__` is set for `<android/api-level.h>` feature guards
- `TLS_IE` vs `TLS_LE` relocation model for thread-local storage

Google's Current Minimum (as of 2026): The Android developer dashboard shows >99% of active devices on API 26+. Google Play's minimum target requirement is API 26. The NDK's default minSdkVersion in new projects is 26.

**ABI stability**: Google guarantees ABI stability for the four above ABIs. The 32-bit x86 and armeabi-v7a ABIs are officially deprecated for new apps but must still compile cleanly for library authors supporting older devices.

**Thumb-2 for ARM32**: `armv7a-linux-androideabi` compiles to Thumb-2 by default. To force full ARM32 mode, pass `-marm`. Most performance-critical code is better served by arm64-v8a; ARM32 exists primarily for devices that predate 64-bit Android.

---

## 236.3 Sysroot Structure and Stub Libraries

The NDK sysroot is the most conceptually distinctive part of the toolchain. Unlike a desktop Linux sysroot that contains the actual `.so` files from the target, the NDK sysroot contains **stub libraries** — `.so` files with symbol tables and `DT_SONAME` entries but no executable code.

```
sysroot/usr/lib/aarch64-linux-android/
├── 21/
│   ├── libc.so        (stub)
│   ├── libm.so        (stub)
│   ├── libdl.so       (stub)
│   └── crtbegin_dynamic.o
├── 23/
│   ├── libc.so        (stub — adds posix_spawn, etc.)
├── 26/
├── ...
├── 35/
│   └── libc.so        (stub — adds all API 35 additions)
└── libc.so -> 35/libc.so  (latest, used when no api specified)
```

Stubs are generated from `*.map.txt` files in the Android platform source tree via the `stubgen` tool. Each `.map.txt` lists every exported symbol with its introduced API level:

```
LIBC {
  global:
    accept;       # introduced=1
    posix_spawn;  # introduced=23
    ...
};
```

At link time the linker checks that every symbol referenced by your `.so` appears in the stub for your minimum API level. If `posix_spawn` is referenced and `minSdkVersion=21`, the link fails with an undefined symbol error. This catch happens at build time, not at runtime — a significant improvement over the old `__attribute__((weak))` runtime-check pattern.

`DT_NEEDED` entries in the final `.so` resolve against the real system libraries at `dlopen` time on device. The stub `.so` is never shipped in the APK.

**Versioned symbols**: Bionic does not use ELF symbol versioning (GNU_VERSYM sections). All symbols are unversioned. This simplifies the stub model but means Bionic cannot provide `GLIBC_2.17`-style compat shims.

---

## 236.4 CMake Integration

The canonical build system for NDK projects is CMake via the `android.toolchain.cmake` toolchain file included in the NDK. Gradle invokes CMake automatically when `externalNativeBuild { cmake { ... } }` is present in `build.gradle`.

### Minimum CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.22)
project(mylib CXX)

add_library(mylib SHARED
    src/mylib.cpp
    src/codec.cpp
)

find_library(log-lib log)        # liblog.so from the NDK
find_library(android-lib android) # libandroid.so

target_link_libraries(mylib
    ${log-lib}
    ${android-lib}
)

target_compile_options(mylib PRIVATE -O3 -fno-exceptions)
```

### Gradle configuration

```groovy
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++20"
                arguments "-DANDROID_STL=c++_shared",
                          "-DANDROID_ARM_NEON=TRUE"
            }
            abiFilters "arm64-v8a", "x86_64"
        }
    }
    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
            version "3.22.1"
        }
    }
}
```

### Key CMake variables

| Variable | Values | Effect |
|----------|--------|--------|
| `ANDROID_ABI` | `arm64-v8a`, `armeabi-v7a`, `x86_64`, `x86` | Target ABI (set by Gradle) |
| `ANDROID_PLATFORM` | `android-21` … `android-35` | min API level |
| `ANDROID_STL` | `c++_shared`, `c++_static`, `none`, `system` | C++ standard library |
| `ANDROID_ARM_NEON` | `TRUE`/`FALSE` | Enable NEON for armeabi-v7a |
| `ANDROID_NATIVE_API_LEVEL` | integer | Alias for ANDROID_PLATFORM |
| `CMAKE_TOOLCHAIN_FILE` | path to `android.toolchain.cmake` | Activates NDK toolchain |

**STL choice**: `c++_static` avoids a runtime `.so` dependency but bloats each `.so`. If multiple `.so` files all link `c++_static`, global state (e.g., `std::locale`, exception tables) is duplicated — a known source of subtle bugs. Prefer `c++_shared` for projects with multiple native libraries, and include `libc++_shared.so` in the APK.

---

## 236.5 Bionic libc Differences

Bionic is not glibc. Porting code written for Linux desktop must account for the following divergences.

### Missing POSIX features

| Feature | Status |
|---------|--------|
| `pthread_cancel` | Not implemented (returns ENOSYS) |
| `execinfo.h` / `backtrace()` | Not in Bionic; use `<unwind.h>` or libunwind |
| `<obstack.h>` | Absent |
| `<rpc/rpc.h>` | Absent |
| `locale_t` | Available API 21+ |
| `open_memstream` | Available API 23+ |
| `getentropy` | Available API 28+ |

`pthread_cancel` is the most common porting surprise. Code that relies on cancellation points must be rewritten using `pthread_testcancel`-equivalent manual check-and-exit patterns or `std::stop_token`.

### Register x18 — shadow call stack

On AArch64 Android, register `x18` is reserved by the platform as the **shadow call stack pointer** when the binary is compiled with `-fsanitize=shadow-call-stack` (the default for platform code since Android 12). Inline assembly or hand-written `.s` files that clobber `x18` will corrupt the shadow stack and produce a crash in `__cfi_slowpath` or simply an SIGSEGV at an unexpected return.

Declare `x18` clobbered in inline assembly constraints, or — for assembly-only hot paths — compile with `-ffixed-x18` to instruct LLVM to never allocate `x18` for register variables:

```cpp
// Correct inline asm on Android AArch64
asm volatile(
    "mov x0, x18\n"   // reading shadow stack ptr (rarely needed)
    : : : "x18"       // clobber x18
);
```

### `__attribute__((visibility("default")))` and `-fvisibility=hidden`

Bionic's dynamic linker (`/system/bin/linker64`) is more aggressive than glibc's `ld.so` in respecting ELF visibility. Build all NDK shared libraries with `-fvisibility=hidden` and explicitly export only the intended API surface with `__attribute__((visibility("default")))` or a `.map` file passed to `--version-script`. Hidden symbols are not resolved across `.so` boundaries, which prevents accidental ABI leakage and speeds up `dlopen`.

### Thread-local storage

Bionic uses ELF TLS (`.tbss`/`.tdata` sections) on AArch64 and x86_64. The old `__thread` emulation via `pthread_getspecific` is no longer needed. Use `thread_local` (C++11) directly; Clang maps it to the TPIDR_EL0-relative model.

---

## 236.6 Multi-ABI APK Layout

An APK is a zip archive. Native libraries reside in `lib/<abi>/`:

```
myapp.apk
├── lib/
│   ├── arm64-v8a/
│   │   ├── libmylib.so
│   │   └── libc++_shared.so
│   ├── armeabi-v7a/
│   │   ├── libmylib.so
│   │   └── libc++_shared.so
│   └── x86_64/
│       ├── libmylib.so
│       └── libc++_shared.so
└── classes.dex
```

The package manager (`pm`) installs only the `.so` files matching the device's primary ABI (and secondary ABI for ARMv8 devices running ARMv7 apps). Including all four ABIs maximises compatibility but increases APK size.

**Android App Bundle (AAB)**: Google Play splits delivery by ABI when the app uses AAB format, shipping only the matching ABI slice to each device. Use `bundletool` locally to verify per-ABI `.apks` output.

**`android:extractNativeLibs="false"`**: Setting this in `AndroidManifest.xml` prevents the package manager from extracting `.so` files to the filesystem. Instead, they are mmap-ed directly from the compressed APK (they must be stored uncompressed, 4 KB page-aligned). This reduces install size and I/O but requires `System.loadLibrary` to work via `/proc/self/fd`-based path, which is supported since API 23.

---

## 236.7 Sanitizers and wrap.sh

The NDK supports Clang's sanitizers with some Android-specific configuration.

### HWAddressSanitizer (HWASan)

HWASan is the preferred memory-safety sanitizer for Android (it is used in production for platform binaries since Android 14 on hardware with MTE). On Armv8 hardware without MTE, it uses top-byte ignore (TBI) to store tag bits in pointer bits 56–59.

```cmake
# Enable HWASan for a target
target_compile_options(mylib PRIVATE -fsanitize=hwaddress -fno-omit-frame-pointer)
target_link_options(mylib PRIVATE -fsanitize=hwaddress)
```

HWASan requires:
- API level 27+ (for the `__hwasan_*` runtime symbols)
- The `libclang_rt.hwasan-aarch64-android.so` runtime from `toolchains/llvm/prebuilt/linux-x86_64/lib/clang/22/lib/linux/`
- `wrap.sh` to preload the runtime on older Android (pre-12)

### wrap.sh mechanism

`wrap.sh` is an Android-specific mechanism for wrapping app process startup. Place a shell script at `lib/<abi>/wrap.sh` and set `android:debuggable="true"` in the manifest. The zygote reads `wrap.sh` and uses it as the interpreter for the app process:

```bash
# lib/arm64-v8a/wrap.sh — preload HWASan runtime
#!/system/bin/sh
LD_HWASAN=1 exec "$@"
```

A more common use is loading ASan/HWASan runtimes that need to be the first library in the process:

```bash
#!/system/bin/sh
LD_PRELOAD=/data/local/tmp/libclang_rt.hwasan-aarch64-android.so exec "$@"
```

`wrap.sh` requires API 27+ and `android:debuggable="true"`. It is not available in release builds.

### AddressSanitizer

ASan is supported but slower than HWASan (2× overhead vs 15%). Use ASan on emulators (x86_64) where HWASan's tag-based approach is emulated.

```bash
aarch64-linux-android35-clang++ -fsanitize=address -fno-omit-frame-pointer \
    -g -O1 -shared -o libfoo.so foo.cpp
```

### UBSan and CFI

```bash
-fsanitize=undefined                # UBSan
-fsanitize=cfi -flto -fvisibility=default  # CFI (requires LTO + default visibility)
```

CFI is enabled by default for Android platform code. It is enforced via `__cfi_slowpath` in `libdl.so`. Third-party `.so` files linked without CFI can still be loaded but will not be CFI-checked on indirect calls from platform code.

---

## 236.8 The Android LLVM Fork and GKI

### Android's LLVM fork

Google maintains a fork of LLVM at `android.googlesource.com/toolchain/llvm-project`. The fork rebases onto upstream LLVM major releases with Android-specific patches:

- AArch64 shadow call stack support (`-fsanitize=shadow-call-stack`)
- Android-specific CFI (`-fsanitize=cfi`)
- Compact unwind tables for Bionic's `_Unwind_Backtrace`
- `hwasan` runtime integrated into the toolchain
- Build system integration (Android.bp → `build/soong`)
- Target-specific tuning for Cortex-X series and Pixel chips

Patches that are production-ready and upstream-relevant are upstreamed; Android-specific patches remain in the fork. The NDK Clang binary is built from this fork.

### Generic Kernel Image (GKI) mandate

Since Android 11, the **Generic Kernel Image** requires all kernel modules to be compiled with Clang (specifically a version from the Android Clang/LLVM prebuilt project). GCC is no longer supported for Android kernel builds.

The GKI Clang requirement means:

```bash
# Building an out-of-tree kernel module for Android
make -C $KERNEL_DIR M=$(pwd) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    CC=clang \
    LLVM=1 \                    # use all LLVM tools (ld.lld, llvm-ar, etc.)
    LLVM_IAS=1 \                # use Clang's integrated assembler
    CLANG_TRIPLE=aarch64-linux-gnu \
    modules
```

`LLVM=1` replaces `$(CC)`, `$(AR)`, `$(LD)`, `$(NM)`, `$(STRIP)` with their LLVM equivalents. `LLVM_IAS=1` uses Clang's integrated assembler instead of the GNU assembler — required because GKI modules must produce `.ko` files binary-compatible with the prebuilt GKI image.

### Clang-built kernel and LTO

Android platform kernels are built with ThinLTO since Android 12 for Pixel devices:

```makefile
# In kernel defconfig
CONFIG_LTO_CLANG_THIN=y
CONFIG_CFI_CLANG=y
```

This enables function-level dead stripping, cross-module inlining, and CFI for the kernel. The resulting `vmlinux` is significantly smaller and CFI-hardened against kernel ROP chains.

---

## Summary

- The NDK ships Clang/LLD wrappers named `<triple><api>-clang`; always use these wrappers, not raw `clang --target=`.
- Four ABI/triple pairs are supported: arm64-v8a, armeabi-v7a, x86_64, x86. API level is embedded in the triple.
- The sysroot uses stub `.so` files generated from `.map.txt` symbol maps; link-time symbol checking prevents API level violations.
- CMake via `android.toolchain.cmake` is the canonical build system; key variables are `ANDROID_ABI`, `ANDROID_PLATFORM`, `ANDROID_STL`.
- Bionic lacks `pthread_cancel`, `execinfo.h`, and several POSIX extensions; AArch64 reserves `x18` for the shadow call stack.
- APKs carry `lib/<abi>/` subtrees; AAB format enables per-ABI delivery splitting.
- HWASan is the preferred sanitizer; `wrap.sh` enables runtime preloading for debugging.
- The Android LLVM fork patches shadow call stack, CFI, HWASan, and compact unwind into the upstream LLVM; the GKI mandate requires Clang for all kernel module builds.
