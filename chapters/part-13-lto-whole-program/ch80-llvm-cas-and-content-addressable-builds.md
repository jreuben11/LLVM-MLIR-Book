# Chapter 80 — llvm-cas and Content-Addressable Builds

*Part XIII — Link-Time and Whole-Program*

The LLVM Content-Addressable Store (llvm-cas) is an infrastructure layer for reproducible, hermetic, and incrementally-efficient builds. Rather than identifying build artifacts by file path and timestamp, CAS identifies them by their cryptographic hash — their **content address**. This approach enables sharing of identical artifacts across build machines and build sessions, safe parallel execution of compilation steps, and elimination of redundant recompilation when inputs have not changed. This chapter covers the CAS object model, integration with Clang for module reuse, the CASFS virtual filesystem, and the CAS plugin API.

## 80.1 Motivation: Problems with Path-Based Builds

### 80.1.1 Non-Determinism in Conventional Builds

Conventional build systems identify artifacts by file path and modification timestamp. This creates several problems:

- **Non-reproducibility**: the same source file compiled at different times or on different machines may produce different output (due to `__DATE__`, `__TIME__`, path-dependent debug info, or environmental variables).
- **Non-shareability**: identical compilation steps on different machines produce separate artifacts even when the output is bitwise identical.
- **Redundant recompilation**: a build system re-runs a compilation step if the input timestamp changes, even if the content is identical (e.g., after `git checkout` and `git checkout -` that restore the same file).
- **Non-hermeticity**: compilation may read from implicit inputs (system headers, environment variables) that are not declared as dependencies, causing stale builds.

### 80.1.2 Content-Addressable Storage

A **content-addressable store** (CAS) solves these problems by:

1. **Uniquely identifying** every artifact by its SHA-256 hash (or similar).
2. **Caching** artifacts in a store keyed by their content hash.
3. **Reusing** cached results whenever inputs hash to the same value.
4. **Sharing** the store across machines (via a remote cache service).

If compilation inputs (source file + all headers + compiler version + flags) hash to the same value, the output can be retrieved from the cache without re-running the compiler.

## 80.2 The LLVM CAS Object Model

### 80.2.1 CAS Objects

The LLVM CAS (`llvm/CAS/` directory) models every compilation artifact as a **CAS object** — an immutable, content-addressed blob or tree:

```
CAS Object:
  id:      CASID (SHA-256 hash of content)
  refs:    [CASID]  (references to other CAS objects this object depends on)
  data:    bytes     (the object's payload)
```

Two types of CAS objects:

- **Blobs**: raw byte content (source files, compiler flags, object code).
- **Trees**: structured collections of `(name, CASID)` pairs, like a filesystem directory.

```cpp
// llvm/include/llvm/CAS/ObjectStore.h
class ObjectStore {
public:
  // Store a blob, get its CASID:
  Expected<ObjectRef> storeFromString(ArrayRef<char> Data);

  // Store a tree (directory):
  Expected<ObjectRef> storeFromRefs(
      ArrayRef<NamedObjectRef> Refs);

  // Retrieve content by CASID:
  Expected<std::optional<ObjectRef>> getReference(const CASID &ID);
  Expected<StringRef> getContent(ObjectRef Ref);
};
```

### 80.2.2 The CASID

A `CASID` is a 256-bit cryptographic hash of the object's content and its referenced object IDs:

```
CASID = SHA-256(data || SHA-256(ref[0]) || SHA-256(ref[1]) || ...)
```

This makes CASIDs **content-derived**: two objects with identical content and the same referenced objects will have the same CASID, regardless of when or where they were created.

### 80.2.3 The On-Disk Store

The default CAS on-disk store uses a directory structure keyed by the first bytes of the CASID:

```
~/.llvm_cas/
  objects/
    ab/
      cd1234ef5678...   (blob: SHA-256 starts with abcd...)
      ef9012ab3456...
    12/
      ...
  index/                 (optional: CASID → file path index)
```

LLD/Clang can be directed to use the CAS store:

```bash
clang -fno-integrated-as -fcas-backend \
      -fcas-fs=my-project-hermetic-fs \
      -cas-path ~/.llvm_cas \
      input.c -o output.o
```

## 80.3 Integration with Clang: Module Reuse

### 80.3.1 C++ Module Caching

C++20 modules (and Clang's `-fmodules`) require building a **Binary Module Interface (BMI)** from a module interface unit (`.cppm`). In large codebases, many translation units depend on the same modules, and the same BMI may be built redundantly by multiple compilation jobs.

CAS integration caches BMIs by their content address:

```bash
# Compile module unit, store BMI in CAS:
clang -std=c++20 -fmodule-output=foo.pcm \
      -fcas-backend -cas-path ~/.llvm_cas \
      foo.cppm

# Import module (reads BMI from CAS if CASID matches):
clang -std=c++20 -fmodule-file=foo.pcm \
      -fcas-backend -cas-path ~/.llvm_cas \
      main.cpp -o main.o
```

When multiple build workers attempt to build the same module, the CAS prevents redundant computation: the first worker stores the BMI; subsequent workers retrieve it from the cache.

### 80.3.2 Action Cache

The **action cache** (`ActionCache` in `llvm/CAS/ActionCache.h`) maps a **cache key** (the hash of all compilation inputs: source, flags, compiler version, system headers) to a **cache result** (the CASID of the compilation output):

```
ActionCache:
  Key = SHA-256(source_CASID || flags_hash || compiler_version || ...)
  Value = CASID of output object

Lookup: ActionCache[key] → CASID
  If CASID found: retrieve object from ObjectStore (cache hit)
  If not found: compile, store output in ObjectStore, store key→CASID mapping
```

This is the **compilation cache** layer: if the same compilation has been performed before (by any machine sharing the cache), the result is retrieved instead of recompiled.

```cpp
// Using the action cache:
llvm::cas::ActionCache AC = getActionCache(CASPath);
llvm::cas::CASID Key = computeCompilationKey(InputFile, CompilerFlags);
auto Result = AC.get(Key);
if (Result)
  return loadFromCAS(*Result);   // cache hit
auto Output = compile(InputFile, CompilerFlags);
AC.put(Key, storeInCAS(Output)); // cache miss: compile and cache
```

## 80.4 CASFS: Hermetic Virtual Filesystem

### 80.4.1 The Problem: Implicit Inputs

A compilation step typically reads files not explicitly listed as dependencies:
- System headers (`/usr/include/string.h`, `/usr/lib/llvm-22/include/llvm/ADT/StringRef.h`)
- Compiler-internal headers (`stddef.h`, `limits.h`)
- Environment variables (`PATH`, `SDKROOT` on macOS)

These **implicit inputs** prevent reproducibility: if a system header changes between two builds, the output changes without the build system detecting it.

### 80.4.2 CASFS: A Content-Addressed Filesystem View

**CASFS** (CAS Filesystem) is a virtual filesystem (`llvm/CAS/CASFS.h`) that presents the compiler with a read-only view of its input files rooted in the CAS:

```
CASFS root (CAS Tree):
  ├── src/
  │   ├── main.cpp   (CASID: abc...)
  │   └── foo.h      (CASID: def...)
  ├── include/       (CASID: ghi...)
  │   └── stddef.h   (system header, snapshotted into CAS)
  └── usr/include/   (CASID: jkl...)
      └── string.h
```

The CASFS tree captures all headers the compiler will read, including system headers, into a single immutable CAS snapshot. The compilation then runs in this hermetic environment: all file reads go through CASFS, which serves content from the CAS store.

```bash
# Create a CASFS snapshot of the compiler's hermetic environment:
clang-cas-build --create-casfs \
    --include-path /usr/include \
    --include-path /usr/lib/llvm-22/include \
    -o hermetic_env.casfs

# Compile in the hermetic environment:
clang -fcas-fs=hermetic_env.casfs \
      -fcas-backend -cas-path ~/.llvm_cas \
      input.c -o output.o
```

All reads from the hermetic filesystem are content-addressed; the build system can compute the compilation's cache key by hashing the CASFS root CASID + compiler flags + source content, without worrying about implicit file changes.

### 80.4.3 Distributed Hermetic Builds

CASFS enables fully distributed builds where compilation workers have no local filesystem access to the source tree — they receive only the CASFS root ID and retrieve all inputs from the shared CAS store:

```
Build coordinator:
  1. Package source files into CASFS tree → root CASID
  2. For each compilation: send (CASID_of_source, compiler_flags, casfs_root)
  3. Workers retrieve inputs from shared CAS, compile, store output in CAS
  4. Coordinator retrieves output CASIDs, links final binary
```

This is similar to how Bazel's remote execution works, but integrated directly into the LLVM build infrastructure.

## 80.5 The CAS Plugin API

### 80.5.1 Plugin Interface

The CAS Plugin API allows external build systems and CI infrastructure to provide custom CAS backends (e.g., a remote distributed cache):

```cpp
// llvm/include/llvm/CAS/CASPluginSession.h
class CASPlugin {
public:
  // Core operations:
  virtual Expected<ObjectRef> storeFromString(StringRef Data) = 0;
  virtual Expected<Optional<ObjectRef>> getReference(CASID ID) = 0;
  virtual Expected<StringRef> getContent(ObjectRef Ref) = 0;

  // Action cache:
  virtual Expected<Optional<CASID>> getActionCache(CASID Key) = 0;
  virtual Error putActionCache(CASID Key, CASID Result) = 0;
};
```

A custom plugin might implement a remote HTTP-based CAS store:

```cpp
class RemoteCASPlugin : public CASPlugin {
  HTTPClient &Client;

  Expected<ObjectRef> storeFromString(StringRef Data) override {
    auto CASIDStr = sha256(Data);
    Client.put("/cas/" + CASIDStr, Data);
    return makeObjectRef(CASIDStr);
  }

  Expected<Optional<ObjectRef>> getReference(CASID ID) override {
    auto Response = Client.get("/cas/" + ID.toString());
    if (Response.status == 404) return std::nullopt;
    return makeObjectRef(ID);
  }
};
```

### 80.5.2 LLVM CAS and Apple's Xcode

Apple's Xcode uses llvm-cas as the foundation for its **Explicit Module Builds** feature (introduced in Xcode 15). Every module compilation is a CAS-keyed operation; the Xcode build system retrieves cached module artifacts from a shared CAS store across team members' machines.

The CAS API is also exposed through the libclang-based Clang C API for build system integration:

```c
// Clang C API for CAS:
CXCASObjectStore store = clang_experimental_cas_OnDiskObjectStore_create(path);
CXCASDatabase db = clang_experimental_cas_getOrCreateObjectStore(store);
// ... use db for compilation caching
clang_experimental_cas_ObjectStore_dispose(store);
```

## 80.6 CAS in the Build Pipeline

### 80.6.1 Integration Points

CAS integrates at three points in the LLVM build pipeline:

1. **Compiler frontend** (Clang): serialization of PCH/PCM files to CAS; `#include` resolution via CASFS.
2. **Compiler backend** (LLVM opt/codegen): caching of optimization pipeline outputs (useful for expensive -O3 transformations).
3. **Linker** (LLD): caching of linking steps when inputs are unchanged.

### 80.6.2 Cache Statistics

The `llvm-cas` tool provides statistics and management commands:

```bash
# Show CAS statistics:
llvm-cas --cas-path ~/.llvm_cas --print-stats

# List cached objects:
llvm-cas --cas-path ~/.llvm_cas --list

# Evict old entries:
llvm-cas --cas-path ~/.llvm_cas --evict --max-size 10g

# Verify CAS integrity:
llvm-cas --cas-path ~/.llvm_cas --verify
```

### 80.6.3 Remote Cache Service

For team-wide sharing, the CAS store can be backed by a remote service implementing the GRPC-based CAS protocol (compatible with Bazel's Remote Execution API):

```bash
# Use remote CAS backend:
clang -fcas-backend \
      -fcas-remote-url=grpc://cas.example.com:1234 \
      -cas-path ~/.llvm_cas \   # local fallback cache
      input.c -o output.o
```

The local CAS acts as an L1 cache; the remote service acts as an L2 cache shared across the team. On cache hit, no compilation occurs; on cache miss, the local CAS stores the result and replicates it to the remote service.

---

## Reproducible Builds and Toolchain Supply-Chain Security

A build is *reproducible* when given identical source inputs it produces bit-for-bit identical outputs regardless of when, where, or by whom it is built. For compiler toolchains, reproducibility is both a correctness property and a security property: it enables independent verification that distributed binaries match their published source. The LLVM CAS infrastructure described in earlier sections of this chapter provides one architectural foundation for reproducibility; this section addresses the broader ecosystem of techniques, standards, and security concerns that surround it.

### The Trusting Trust Problem

Ken Thompson's 1984 Turing Award lecture, "Reflections on Trusting Trust," posed a question that remains the foundational challenge of toolchain security: can you trust a compiler? Thompson described a compiler modified to inject a backdoor into the login program and simultaneously to perpetuate itself in future compilations of the compiler — even after the "malicious" source code was removed. The attack survives because the compiled compiler binary carries the Trojan; auditing the source is insufficient if the compiler that compiles that source is already compromised.

The formal statement: you cannot trust a binary without trusting every tool in the chain that produced it — the compiler, the assembler, the linker, the operating system, all the way back to the hardware. This chain-of-trust problem is not merely theoretical.

The **XZ utils attack** (2024, CVE-2024-3094) demonstrated that Thompson's scenario translates directly to modern software supply chains. A malicious maintainer, operating under the pseudonym "Jia Tan," spent two years building trust in the XZ compression library project before embedding a carefully obfuscated backdoor in the build system. The backdoor specifically targeted SSH daemon authentication in systemd-linked distributions. Because XZ is shipped by essentially all major Linux distributions, the attack would have given the attacker remote access to a significant fraction of the world's Linux servers. It was discovered by accident — a Microsoft engineer noticed abnormal latency in SSH login times on Debian Sid.

The attack's structure is precisely Thompsonian: the backdoor was not in the library source but in the build artifacts and the autoconf machinery. Supply-chain attacks — compromising a dependency rather than the target software directly — are now the dominant class of software security threat, accounting for a rapidly growing fraction of CVEs and incident reports. The SLSA framework (Supply-chain Levels for Software Artifacts) was developed specifically to categorize and mitigate this threat class.

The response to the trusting trust problem has two prongs: **bootstrappable builds** (described below) provide a path to establishing trust in a compiler from a minimal auditable seed; **reproducible builds** provide a mechanism for independent verification of produced binaries against their claimed source.

### Reproducibility Requirements for Compilers

A compiler introduces non-determinism through a surprising variety of mechanisms. Eliminating them requires understanding each source:

**Timestamps.** The C preprocessor macros `__DATE__` and `__TIME__` expand to the compilation timestamp; debug information (DWARF `DW_AT_comp_dir`, PE/COFF timestamps, embedded build timestamps in Mach-O load commands) likewise captures the build time. The standard fix is the `SOURCE_DATE_EPOCH` environment variable, introduced by the Debian reproducible-builds project and now supported by GCC (since 4.9), Clang (since Clang 8), LLD, GNU Binutils, and most language toolchains. When `SOURCE_DATE_EPOCH` is set, all timestamp emissions use this fixed Unix epoch value rather than the current time:

```bash
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
clang -c input.c -o output.o   # DW_AT_comp_dir timestamps use SOURCE_DATE_EPOCH
```

Clang's implementation is in `clang/lib/Driver/ToolChains/Clang.cpp` and `clang/lib/Frontend/CompilerInvocation.cpp`; the epoch is threaded through to `CodeGenOptions.SourceDateEpoch`.

**Filesystem ordering.** `readdir()` returns entries in inode order, which is filesystem-dependent and may vary between ext4 and XFS, between Linux kernel versions, or between build machines. Build systems that glob directories rather than listing explicit inputs inherit this non-determinism. Fixes: always sort input file lists explicitly; avoid `*.cpp` globs without sorting. LLD's `--reproduce` flag (see below) captures a reproducible replay of all linker inputs.

**Hash map iteration order.** `std::unordered_map` and `std::unordered_set` iteration order is implementation-defined and can vary with the memory allocator's address layout. The LLVM coding standard explicitly bans `std::unordered_map` and `std::unordered_set` in LLVM code for this reason; the approved alternative is `llvm::DenseMap`, which as of LLVM 8 produces deterministic ordering on destruction (the backing array is scanned linearly). This policy is documented in `llvm/docs/CodingStandards.rst`.

**Debug information paths.** `DW_AT_comp_dir` embeds the absolute build directory path; `DW_AT_name` may embed source file paths. On two different build machines with different home directory layouts (`/home/alice/build` vs. `/home/bob/work`), identical source produces different debug info. Fixes:

```bash
clang -fdebug-prefix-map=/home/alice/build=. \
      -fmacro-prefix-map=/home/alice/build=. \
      -c input.c -o output.o
```

`-fdebug-prefix-map=old=new` rewrites `DW_AT_comp_dir` and `DW_AT_name` entries; `-fmacro-prefix-map=old=new` rewrites paths embedded via `__FILE__`. Using `.` as the replacement produces paths relative to the source root, which are independent of build location.

**PGO profile paths.** Instrumented binaries record the absolute path of the profraw file to which profile data is written. When PGO data is gathered on build machine A and used on build machine B, the embedded paths differ. Fix: `-fprofile-prefix-map=old=new` (Clang 12+), analogous to `-fdebug-prefix-map`.

**Address space layout.** PIE (position-independent executable) binaries linked with randomized base addresses embed load addresses in metadata that varies. Fix: use a reproducible base address during builds (not at runtime, which still uses ASLR), or use relative relocations throughout.

**Parallel compilation ordering.** When multiple translation units are compiled in parallel, the order in which object files are emitted to the linker can vary between runs if the build system does not sort its inputs. LLD processes object files in the order they are passed; sorting the input list before invocation ensures determinism.

**Thread scheduling within a single compilation.** Clang's parallel code generation (ThinLTO's parallel backend) can produce different output if thread scheduling varies. The fix is ensuring that parallel outputs are sorted before merging.

### SOURCE_DATE_EPOCH and Clang's Reproducibility Flags

A fully reproducible Clang invocation for a project built from a git repository:

```bash
# Set SOURCE_DATE_EPOCH to the timestamp of the last git commit:
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)

# Compile with all reproducibility flags:
clang -fdebug-prefix-map=$(pwd)=. \
      -fmacro-prefix-map=$(pwd)=. \
      -fprofile-prefix-map=$(pwd)=. \
      -Wno-builtin-macro-redefined \
      -D__DATE__="" -D__TIME__="" \
      -c input.c -o output.o
```

The `-D__DATE__=""` and `-D__TIME__=""` approach suppresses timestamp macros entirely; the alternative is to let `SOURCE_DATE_EPOCH` handle them, which produces fixed strings (`"Jan  1 1970"`, `"00:00:00"`) rather than empty strings.

For LLD, the `--reproduce` flag captures all linker inputs into a tar archive that can be replayed to reproduce the link step on any machine:

```bash
ld.lld --reproduce /tmp/linker-repro.tar \
       -o output input1.o input2.o -lc
# Later, on any machine:
tar xf /tmp/linker-repro.tar
ld.lld @response-file-from-tar
```

This is invaluable for diagnosing linker bugs and for verifying that a link step is deterministic.

For PGO-instrumented builds where profiles are collected and then used on a different machine:

```bash
# Instrument phase (machine A):
clang -fprofile-generate=/tmp/profiles input.c -o instrumented

# Use phase (machine B, with path remapping):
clang -fprofile-use=/tmp/profiles/default.profdata \
      -fprofile-prefix-map=/home/machineA/build=. \
      input.c -o optimized
```

### llvm-cas as a Reproducibility Primitive

The LLVM CAS infrastructure described in sections 80.1–80.6 provides a formal basis for reproducibility verification. The core insight: if two independent compilations of the same source produce the same CASID for the output object, those objects are provably identical — the content hash is the proof. CAS-keyed builds therefore make reproducibility *structurally guaranteed* rather than empirically tested.

Content-addressable storage provides several concrete reproducibility guarantees:

**Hermetic environments via CASFS.** As described in section 80.4, CASFS presents the compiler with a read-only view of all inputs (source files, headers, system headers) rooted in a CAS tree. Because every file read goes through the CAS, implicit input dependencies are eliminated: the CASFS root CASID is a complete, verifiable description of the compiler's input environment. Two compilations with the same CASFS root CASID, compiler binary, and flags are guaranteed to produce the same output.

**Cache correctness via CASID.** The action cache (`ActionCache`) stores the mapping `key → CASID_of_output`. When a remote build cache returns a cached result, the returned CASID can be verified against a locally recomputed CASID, providing tamper evidence. A malicious cache cannot substitute a different object without detection.

**Distributed build verification.** In a team build using a shared CAS store, any team member can independently re-run a compilation and verify that the resulting CASID matches the cached entry. This is the CAS-based equivalent of the reproducible builds project's independent verification.

Inspecting the CAS store to verify build outputs:

```bash
# Print the CAS tree rooted at a known CASID:
llvm-cas --cas-path ~/.llvm_cas \
         --print-tree sha256:abc123def456...

# Verify that a locally-built object matches the cached CASID:
llvm-cas --cas-path ~/.llvm_cas \
         --verify sha256:abc123def456...

# List all objects in the CAS:
llvm-cas --cas-path ~/.llvm_cas --list
```

The combination of `SOURCE_DATE_EPOCH`, prefix maps, sorted inputs, and CASFS hermetic environments addresses all known sources of non-determinism in Clang/LLD builds. The CASID of the final output becomes a verifiable attestation that the build was reproducible.

### Verifying Reproducibility: diffoscope

Once a build claims to be reproducible, the claim must be verified. The standard tool is **diffoscope**, developed by the Debian Reproducible Builds project. Unlike `diff`, diffoscope understands binary formats — it recursively unpacks ELF binaries, inspects DWARF sections, reads archive member headers, decompresses ZIP/JAR contents, and produces human-readable diffs at the level of semantic differences rather than raw bytes.

```bash
# Compare two builds of the same library:
diffoscope build1/libfoo.a build2/libfoo.a

# Typical output on a non-reproducible build:
# ├── libfoo.a
# │   ├── foo.o
# │   │   ├── .debug_info
# │   │   │   ├── DW_AT_comp_dir
# │   │   │   │   ● /home/alice/build vs /home/bob/build
# │   ├── archive header timestamp
# │   │   ● 1716489600 vs 1716576000
```

Common findings from diffoscope analysis:

- **Archive header timestamps**: `ar` archive member headers include a modification timestamp. Fix: pass `--deterministic` to `ar` (or `llvm-ar --deterministic`), which zeros all timestamps.
- **Debug info paths**: `DW_AT_comp_dir`, `DW_AT_name` containing build-directory-absolute paths — fixed by `-fdebug-prefix-map`.
- **Section ordering**: LLD outputs sections in a deterministic order since LLD 9; older linkers may not. Fix: pin to LLD 9+ or sort section contributions explicitly in the linker script.
- **Embedded timestamps**: DWARF `DW_AT_language_version` or vendor extensions sometimes embed build dates — requires patching.

LLVM's own CI includes a reproducibility check: `ninja check-llvm` must produce deterministic output across repeated runs on the same machine. The LLVM test suite includes explicit tests in `llvm/test/Assembler/` and `clang/test/Driver/` that verify `SOURCE_DATE_EPOCH` handling.

For automated verification in CI, the pattern is:

```bash
# Build twice with identical inputs:
cmake -DSOURCE_DATE_EPOCH=$(git log -1 --format=%ct) ... && ninja -j$(nproc)
cp build/lib/libLLVM.so build1/libLLVM.so
ninja clean && ninja -j$(nproc)
cp build/lib/libLLVM.so build2/libLLVM.so

# Verify:
if ! cmp build1/libLLVM.so build2/libLLVM.so; then
    diffoscope build1/libLLVM.so build2/libLLVM.so
    exit 1
fi
```

### Bootstrappable Builds

Reproducible builds address *verification* of binary outputs against source; bootstrappable builds address *trust* in the toolchain itself. The bootstrappable builds project (bootstrappable.org) aims to build the entire software stack — from a tiny, auditable seed — through a sequence of compilation stages, each built by the previous stage.

The motivation is the trusting trust problem: if you trust a GCC binary distributed by your Linux distribution, you are trusting every engineer who touched GCC and every system that compiled it, going back decades. The bootstrappable approach breaks this chain by making every stage auditable.

The standard bootstrap path proceeds through these stages:

- **Stage 0**: `hex0` or `M0` — a minimal hex assembler, typically 357 bytes of machine code, written in hexadecimal and auditable by hand. This is the trust anchor: a human can verify this binary by inspection.
- **Stage 1**: `M1` macro assembler, bootstrapped from `hex0`. A few kilobytes of hex.
- **Stage 2**: `cc0` — a minimal C compiler subset, written in M1 assembly. Compiles a very restricted C dialect.
- **Stage 3**: `tcc` (Tiny C Compiler), compiled by `cc0`. Supports most of C89.
- **Stage 4**: `mes` (Mes Scheme), and `mescc` — a C compiler subset in Scheme.
- **Stage 5–N**: Build GCC, then Clang/LLVM, from source using progressively more capable compilers.

**GNU Guix** takes this approach furthest: the entire Guix package collection is built from source, with the trust anchor being a small bootstrap binary (`guix-bootstrap`) committed to the git repository and auditable. `guix build --check` re-builds a package and verifies that the output matches the cached derivation, making reproducibility a first-class property of the package manager.

**NixOS/Nix** similarly provides reproducible builds: `nix-build --check` re-builds and byte-compares against the cached Nix store path. The Nix store is content-addressed (using SHA-256 of inputs in "input-addressed" mode, or SHA-256 of outputs in "content-addressed" mode), structurally similar to LLVM CAS.

**The LLVM bootstrap build** is a practical reproducibility test at the toolchain level. A stage1→stage2→stage3 build (`-DLLVM_BUILD_RUNTIME=ON`, `-DBOOTSTRAP_LLVM_ENABLE_LTO=ON`) compiles Clang with the host compiler (stage1), then compiles Clang again with stage1 (stage2), then compiles Clang again with stage2 (stage3). If stage2 and stage3 produce identical binaries, the build is self-hosting and reproducible — a strong indication that the compiler is free of Thompsonian backdoors. The LLVM release process requires a successful stage3 bootstrap for all tier-1 targets:

```bash
cmake -G Ninja -DCLANG_ENABLE_BOOTSTRAP=ON \
      -DBOOTSTRAP_LLVM_ENABLE_LTO=Thin \
      ../llvm
ninja clang-bootstrap-deps
ninja stage2     # builds with stage1 clang
ninja stage2-check-all
# Optionally: ninja stage3 && diff stage2/bin/clang stage3/bin/clang
```

### SBOMs and Signed Toolchain Releases

The third pillar of toolchain supply-chain security is *attestation*: producing machine-readable records of what was built, how, and from what inputs, so that downstream consumers can reason about the provenance of the tools they use.

**Software Bill of Materials (SBOM).** An SBOM is a structured list of all components that comprise a software artifact: source repositories, versions, licenses, and dependency graphs. Two standard formats dominate:

- **SPDX** (Software Package Data Exchange, ISO/IEC 5962:2021): XML, JSON, or tag-value text; the Linux Foundation's format; preferred by the Linux kernel community.
- **CycloneDX**: JSON/XML; OWASP project; richer vulnerability-tracking metadata.

NTIA (National Telecommunications and Information Administration) has published minimum elements for an SBOM: supplier name, component name, version, component hash, dependency relationships, and the SBOM author and timestamp. For a compiled binary, the SBOM should identify the compiler version (`clang --version` output), all library dependencies, and the source commit hash:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "components": [
    {
      "type": "library",
      "name": "llvm",
      "version": "22.1.3",
      "hashes": [{"alg": "SHA-256", "content": "abc123..."}],
      "externalReferences": [{"type": "vcs",
        "url": "https://github.com/llvm/llvm-project/tree/llvmorg-22.1.3"}]
    }
  ]
}
```

SBOMs enable automated vulnerability tracking: when a CVE is published against a component (e.g., a buffer overflow in a specific version of `libz`), tools like `grype` or `osv-scanner` can scan SBOMs to identify affected binaries.

**Signed LLVM releases.** LLVM release artifacts are distributed with GPG signatures from the release manager:

```bash
# Download and verify an LLVM source tarball:
curl -LO https://github.com/llvm/llvm-project/releases/download/llvmorg-22.1.0/llvm-project-22.1.0.src.tar.xz
curl -LO https://github.com/llvm/llvm-project/releases/download/llvmorg-22.1.0/llvm-project-22.1.0.src.tar.xz.sig
gpg --keyserver keyserver.ubuntu.com --recv-keys <release-manager-key-id>
gpg --verify llvm-project-22.1.0.src.tar.xz.sig llvm-project-22.1.0.src.tar.xz
```

The release manager's GPG key fingerprint is published on the LLVM website and in the release notes. Binary packages for distributions (Debian, Fedora, Ubuntu) are signed by the distribution's package signing keys, which are separately managed.

**Sigstore and SLSA.** The emerging industry standard for supply-chain attestation combines two systems:

*Sigstore* is a keyless signing infrastructure designed for CI/CD pipelines. Instead of managing long-lived GPG keys, Sigstore uses OIDC (OpenID Connect) tokens issued by CI providers (GitHub Actions, GitLab CI, Google Cloud) to generate short-lived signing certificates anchored to a public transparency log (Rekor). A GitHub Actions workflow can sign a release artifact with no key management:

```yaml
# In GitHub Actions, using the cosign action:
- uses: sigstore/cosign-installer@v3
- run: |
    cosign sign-blob --yes \
      --bundle output.bundle \
      build/libLLVM.so
```

Consumers verify with:

```bash
cosign verify-blob --bundle output.bundle \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp "^https://github.com/llvm/llvm-project/" \
  build/libLLVM.so
```

*SLSA* (Supply-chain Levels for Software Artifacts, pronounced "salsa") is a framework that defines four levels of supply-chain integrity:

| Level | Requirement |
|-------|-------------|
| SLSA 1 | Build process documented; SBOM generated |
| SLSA 2 | Signed provenance; version-controlled build definition |
| SLSA 3 | Hermetic build; no persistent credentials in build environment |
| SLSA 4 | Two-person review; reproducible builds verified |

SLSA Level 3 specifically requires *hermetic builds* — exactly the property that CASFS provides for Clang builds. SLSA Level 4 requires *reproducible builds* — the property that `SOURCE_DATE_EPOCH` and CAS-keyed compilation provide. The LLVM project's GitHub Actions release pipelines are moving toward SLSA Level 3 compliance.

**Reproducible Builds project tracking.** The Reproducible Builds project (reproducible-builds.org) maintains a public database tracking reproducibility status of packages in major distributions. The LLVM toolchain packages in Debian (`llvm-22`, `clang-22`, `lld-22`) are tracked at [tests.reproducible-builds.org/debian](https://tests.reproducible-builds.org/debian/), which shows the percentage of packages that produce bit-identical output across two independent builds with different build paths, usernames, and timestamps. As of LLVM 22, the Clang and LLD packages in Debian are fully reproducible; the remaining non-reproducible packages typically involve generated documentation with embedded timestamps.

The full supply-chain security stack for a production LLVM-based toolchain therefore comprises: CASFS hermetic compilation (eliminating implicit inputs), `SOURCE_DATE_EPOCH` and prefix maps (eliminating timestamp and path non-determinism), CAS-keyed action caches (providing content-addressed verification of cached results), SLSA-attested CI builds (providing provenance records), Sigstore-signed release artifacts (providing keyless, auditable signatures), and SBOM generation (providing component-level vulnerability tracking).

---

## Chapter Summary

- **LLVM CAS** (Content-Addressable Store) identifies every compilation artifact by a SHA-256 content hash (`CASID`), enabling reproducible, hermetic, and incrementally-efficient builds independent of file paths and timestamps.
- **CAS objects** are immutable blobs or trees referenced by their CASID; the on-disk store is a directory of content-hashed files; remote stores share artifacts across machines.
- **Clang integration** uses the CAS to cache PCH/PCM module artifacts via an **action cache** (`ActionCache[key] → CASID`): compilation is skipped when all inputs hash to a previously-seen key.
- **CASFS** presents the compiler with a hermetic, read-only virtual filesystem rooted in a CAS tree that captures all inputs (source, headers, system headers) as content-addressed snapshots, eliminating implicit input dependencies.
- **The CAS Plugin API** allows custom backends (remote HTTP, GRPC Remote Execution API, Apple's Xcode network cache) to implement the `CASPlugin` interface, integrating with LLVM's build infrastructure.
- Practical applications: Apple's Xcode Explicit Module Builds, distributed hermetic CI builds where workers receive only CASID inputs and produce CASID outputs, and team-shared compilation caches eliminating redundant work across developers.


---

@copyright jreuben11
