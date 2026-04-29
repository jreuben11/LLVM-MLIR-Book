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

## Chapter Summary

- **LLVM CAS** (Content-Addressable Store) identifies every compilation artifact by a SHA-256 content hash (`CASID`), enabling reproducible, hermetic, and incrementally-efficient builds independent of file paths and timestamps.
- **CAS objects** are immutable blobs or trees referenced by their CASID; the on-disk store is a directory of content-hashed files; remote stores share artifacts across machines.
- **Clang integration** uses the CAS to cache PCH/PCM module artifacts via an **action cache** (`ActionCache[key] → CASID`): compilation is skipped when all inputs hash to a previously-seen key.
- **CASFS** presents the compiler with a hermetic, read-only virtual filesystem rooted in a CAS tree that captures all inputs (source, headers, system headers) as content-addressed snapshots, eliminating implicit input dependencies.
- **The CAS Plugin API** allows custom backends (remote HTTP, GRPC Remote Execution API, Apple's Xcode network cache) to implement the `CASPlugin` interface, integrating with LLVM's build infrastructure.
- Practical applications: Apple's Xcode Explicit Module Builds, distributed hermetic CI builds where workers receive only CASID inputs and produce CASID outputs, and team-shared compilation caches eliminating redundant work across developers.


---

@copyright jreuben11
