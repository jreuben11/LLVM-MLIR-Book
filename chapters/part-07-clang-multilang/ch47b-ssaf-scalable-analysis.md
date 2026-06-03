# Chapter 47b — Scalable Static Analysis with SSAF

*Part VII — Clang Tooling and Heterogeneous Compilation*

## Table of Contents

- [47b.1 The Whole-Program Problem](#47b1-the-whole-program-problem)
- [47b.2 Architecture Overview](#47b2-architecture-overview)
  - [47b.2.1 The Three-Component Pipeline](#47b221-the-three-component-pipeline)
  - [47b.2.2 Analogy to ThinLTO](#47b22-analogy-to-thinlto)
  - [47b.2.3 Position in the Compilation Pipeline](#47b23-position-in-the-compilation-pipeline)
- [47b.3 The SSAF Data Model: Entities, Names, and Namespaces](#47b3-the-ssaf-data-model-entities-names-and-namespaces)
  - [47b.3.1 EntityName: Stable Cross-TU Identity](#47b31-entityname-stable-cross-tu-identity)
  - [47b.3.2 BuildNamespace: Compilation Units and Link Units](#47b32-buildnamespace-compilation-units-and-link-units)
  - [47b.3.3 EntityId and EntityIdTable: Efficient Interning](#47b33-entityid-and-entityidtable-efficient-interning)
  - [47b.3.4 SummaryName: Keying Analysis Results](#47b34-summaryname-keying-analysis-results)
  - [47b.3.5 Mapping AST Declarations to Entities](#47b35-mapping-ast-declarations-to-entities)
- [47b.4 Summary Format and Serialization](#47b4-summary-format-and-serialization)
  - [47b.4.1 Per-Function Summary Data Model](#47b41-per-function-summary-data-model)
  - [47b.4.2 JSON Serialization Architecture](#47b42-json-serialization-architecture)
  - [47b.4.3 Incremental Analysis and Caching](#47b43-incremental-analysis-and-caching)
- [47b.5 Per-TU Analysis Phase](#47b5-per-tu-analysis-phase)
  - [47b.5.1 What the Per-TU Pass Analyzes](#47b51-what-the-per-tu-pass-analyzes)
  - [47b.5.2 Integration with Compilation Databases](#47b52-integration-with-compilation-databases)
  - [47b.5.3 Boundary with the Path-Sensitive Analyzer](#47b53-boundary-with-the-path-sensitive-analyzer)
- [47b.6 Whole-Program Linking Phase](#47b6-whole-program-linking-phase)
  - [47b.6.1 Bottom-Up Callgraph Traversal](#47b61-bottom-up-callgraph-traversal)
  - [47b.6.2 The Checker Transfer Function Interface](#47b62-the-checker-transfer-function-interface)
  - [47b.6.3 Built-in Cross-TU Checkers](#47b63-built-in-cross-tu-checkers)
  - [47b.6.4 False Positive Rate Tradeoffs](#47b64-false-positive-rate-tradeoffs)
- [47b.7 Writing an SSAF Checker](#47b7-writing-an-ssaf-checker)
  - [47b.7.1 The Checker Extension Points](#47b71-the-checker-extension-points)
  - [47b.7.2 Example: Cross-TU Null Propagation Checker](#47b72-example-cross-tu-null-propagation-checker)
  - [47b.7.3 Linking the Checker into the Build](#47b73-linking-the-checker-into-the-build)
- [47b.8 Integration with Build Systems](#47b8-integration-with-build-systems)
  - [47b.8.1 CMake Integration](#47b81-cmake-integration)
  - [47b.8.2 Ninja Phony Targets](#47b82-ninja-phony-targets)
  - [47b.8.3 CI Parallelism Pattern](#47b83-ci-parallelism-pattern)
- [47b.9 Comparison with Other Whole-Program Approaches](#47b9-comparison-with-other-whole-program-approaches)
- [47b.10 Practical Example: Cross-Module Use-After-Free](#47b10-practical-example-cross-module-use-after-free)
  - [47b.10.1 The Scenario](#47b101-the-scenario)
  - [47b.10.2 How Summary Facts Propagate](#47b102-how-summary-facts-propagate)
  - [47b.10.3 Producing the Diagnostic](#47b103-producing-the-diagnostic)
- [47b.11 Chapter Summary](#47b11-chapter-summary)

---

## 47b.1 The Whole-Program Problem

The path-sensitive static analyzer described in [Chapter 45 — The Static Analyzer](ch45-the-static-analyzer.md) is one of the most precise bug-finding tools in the open-source compiler ecosystem. By threading symbolic state through every feasible control-flow path in a function, it can prove that a null dereference occurs on a specific call chain that no test suite exercises. The cost of that precision is the state-space explosion inherent to symbolic execution: analyzing a single complex function can produce tens of thousands of `ExplodedNode` states, and inlining callees multiplies that budget further. As a consequence, clang's built-in analyzer imposes hard limits — `InlineMaxStackDepth`, `maxBlockVisitOnPath` — and refuses to follow call chains that cross translation-unit (TU) boundaries in the same analysis run.

The numbers make the scaling problem concrete. A modern systems project at the scale of Chromium, LLVM itself, or the Linux kernel contains on the order of 10–50 million lines of code spread across thousands of translation units. The clang static analyzer, running with a budget of 128K `ExplodedNode` states per function and inlining up to 8 levels deep, analyzes perhaps a few thousand functions per minute on a fast workstation. A full per-TU symbolic execution of a 10M-LOC project takes tens of CPU-hours — feasible in CI if distributed, but only with per-TU isolation that prevents cross-TU reasoning. Extending that analysis to track a pointer from its allocation site in `liballoc.so` through a global data structure in `libregistry.so` to a use site in the main executable is qualitatively out of reach for unmodified symbolic execution.

This limitation matters because large-scale security bugs do not respect TU boundaries. The class of vulnerabilities that killed real production systems — heap use-after-free through confused ownership across library boundaries, null dereferences on return values from APIs documented as "may return null" but used without checking in callers, memory leaks caused by mismatched allocator/deallocator pairs split across shared libraries — are structurally invisible to per-TU analysis. The clang CTU extension (see §47b.9) solves this by re-parsing foreign TUs on demand, but at a cost that makes it impractical for projects larger than a few hundred thousand lines.

The Scalable Static Analysis Framework (SSAF) addresses this by adopting a **summary-based, link-time architecture**: each translation unit is analyzed independently to produce a compact summary of observable program facts, and a separate link-time step aggregates all summaries to check whole-program properties. SSAF ships in LLVM 22 as the foundational data model in `clang/Analysis/Scalable/`, implemented in `libclangAnalysisScalable.a`. The per-TU emission tooling and link-time aggregation binary are evolving capabilities being built on top of this core model; the sections below describe the design as it exists in LLVM 22.1.x and the direction in which the framework is growing.

---

## 47b.2 Architecture Overview

### 47b.2.1 The Three-Component Pipeline

SSAF decomposes whole-program analysis into three cooperating components:

1. **Per-TU analysis pass** (`clang-ssaf-analyzer`): A Clang frontend action that runs alongside compilation, inspects each function in the TU, and emits an analysis summary file — conventionally `<basename>.ssaf.json` — alongside the object file. The pass does not perform symbolic execution; it extracts structural facts from the AST and light dataflow.

2. **Summary format (Link Unit Summary)**: The on-disk and in-memory representation of per-function analysis results, keyed by stable cross-TU identifiers so that the link-time step can correlate entries from different object files without access to AST state.

3. **Link-time analysis pass** (`clang-ssaf-linker`): Reads all summary files produced during compilation, builds a whole-program callgraph, and runs checker logic that requires cross-TU visibility — use-after-free across module boundaries, null propagation through API surfaces, and allocation/deallocation imbalances in multi-library programs.

### 47b.2.2 Analogy to ThinLTO

The analogy to ThinLTO ([Chapter 77 — LTO and ThinLTO](../part-13-lto-whole-program/ch77-lto-and-thinlto.md)) is precise. ThinLTO splits interprocedural optimization into two passes: `clang` generates per-TU function summary bitcode (stored in the `.bc` file's `GLOBALVAL_SUMMARY_BLOCK`), and `lld` reads all summaries to guide cross-TU inlining and dead-code elimination without re-parsing every source file. SSAF applies the same pattern to static analysis: the per-TU pass produces analysis summaries cheap enough to be regenerated at compile time, and the link-time pass performs analysis — not optimization — using the aggregated summaries.

The structural parallel extends to the identity mechanism. ThinLTO uses LLVM IR value GUIDs (hashes of mangled names) as stable cross-module function identifiers; SSAF uses Clang USRs wrapped in `EntityName`. Both schemes solve the same problem — two separately compiled modules need to refer to the same function without sharing AST or IR state — using the canonical identifier for their respective representation level.

The key property that makes this scalable is **independence**: per-TU summaries can be generated in parallel with the same parallelism as object compilation, and the link-time step's compute cost is proportional to the summary size (linear in the number of public APIs), not to the program's total control-flow complexity. A ThinLTO summary for a 100K-function library is on the order of a few megabytes; an SSAF summary for the same library, recording callgraph edges and ownership facts but not symbolic path conditions, is expected to be of comparable size.

### 47b.2.3 Position in the Compilation Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│  Build system (CMake / Ninja / Bazel)                           │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐     ┌─────────────┐          │
│  │  clang      │  │  clang      │ ... │  clang      │          │
│  │  foo.cpp    │  │  bar.cpp    │     │  baz.cpp    │          │
│  │  → foo.o    │  │  → bar.o   │     │  → baz.o    │          │
│  │  → foo.ssaf │  │  → bar.ssaf│     │  → baz.ssaf │          │
│  └──────┬──────┘  └──────┬─────┘     └──────┬──────┘          │
│         │                │                   │                 │
│         └────────────────┴─────────────────--┘                 │
│                                    │                           │
│                     ┌──────────────▼──────────────┐            │
│                     │   clang-ssaf-linker          │            │
│                     │   (aggregates summaries,     │            │
│                     │    runs whole-program checks) │           │
│                     └─────────────────────────────┘            │
│                                    │                           │
│                          Diagnostic report                      │
└─────────────────────────────────────────────────────────────────┘
```

The analysis pipeline runs in parallel with linking; the `.ssaf.json` files are build artifacts with the same lifetime as `.o` files and participate in the same incremental dependency tracking.

---

## 47b.3 The SSAF Data Model: Entities, Names, and Namespaces

The foundational layer of SSAF that ships in LLVM 22.1.x is a data model for tracking program entities across compilation boundaries. All headers live under
[`clang/Analysis/Scalable/`](https://github.com/llvm/llvm-project/blob/main/clang/include/clang/Analysis/Scalable/)
and compile into `libclangAnalysisScalable.a`; the namespace for all types is `clang::ssaf`.

### 47b.3.1 EntityName: Stable Cross-TU Identity

The central design challenge in any summary-based analysis is **identity**: how does the link-time aggregator know that the `malloc` return value recorded in `foo.ssaf.json` is the same entity as the pointer that appears in a use site recorded in `bar.ssaf.json`? Mangled C++ names collide across namespaces and are template-instantiation-sensitive; source locations are TU-local. SSAF uses Unified Symbol References (USRs) as the stable identifier substrate, wrapped in a richer type:

```cpp
// clang/Analysis/Scalable/Model/EntityName.h
namespace clang::ssaf {

class EntityName {
  std::string USR;           // Clang USR: globally unique, stable across TUs
  llvm::SmallString<16> Suffix; // Distinguishes sub-entities of the same decl
  NestedBuildNamespace Namespace; // Build-system qualification

  auto asTuple() const { return std::tie(USR, Suffix, Namespace); }

public:
  // Constructed only by ASTEntityMapping helpers, not by client code.
  EntityName(llvm::StringRef USR, llvm::StringRef Suffix,
             NestedBuildNamespace Namespace);

  bool operator==(const EntityName &) const;
  bool operator<(const EntityName &) const;

  EntityName makeQualified(NestedBuildNamespace Namespace) const;

  friend class LinkUnitResolution;  // link-time aggregation layer
  friend class SerializationFormat; // JSON emission layer
};

} // namespace clang::ssaf
```

The comment in the header states the design goal precisely: "EntityName provides a globally unique identifier for program entities that **remains stable across compilation boundaries**. This enables whole-program analysis to track and relate entities across separately compiled translation units." The `Suffix` field differentiates sub-entities sharing a USR — for example, the return value of a function versus its first parameter both derive from the same function USR but carry distinct suffix tags.

### 47b.3.2 BuildNamespace: Compilation Units and Link Units

The `NestedBuildNamespace` represents the hierarchical context in which an entity was compiled. This directly models the ThinLTO notion of a link unit: entities discovered within a compilation unit may be promoted into a link unit when that `.o` file is linked into a shared library or executable.

```cpp
// clang/Analysis/Scalable/Model/BuildNamespace.h
namespace clang::ssaf {

enum class BuildNamespaceKind : unsigned short {
  CompilationUnit,  // entity belongs to a single TU (.o file)
  LinkUnit          // entity has been promoted to a link unit (.so / executable)
};

class BuildNamespace {
  BuildNamespaceKind Kind;
  std::string Name; // e.g., "libfoo.so" or "src/foo.cpp"

public:
  static BuildNamespace makeCompilationUnit(llvm::StringRef CompilationId);
};

class NestedBuildNamespace {
  std::vector<BuildNamespace> Namespaces; // ordered: CU first, then LU

public:
  static NestedBuildNamespace makeCompilationUnit(llvm::StringRef CompilationId);

  // Append additional namespace qualification.
  NestedBuildNamespace makeQualified(NestedBuildNamespace Namespace) const;
};

} // namespace clang::ssaf
```

The header comment explains the layered intent: "an entity might be qualified by a compilation unit namespace followed by a shared library namespace." This is the SSAF equivalent of ThinLTO's `FunctionSummary::GVFlags::Linkage`: a function analyzed in isolation inside `foo.cpp` carries a `CompilationUnit` namespace; when the aggregator processes the link step, it promotes visible functions to the `LinkUnit` namespace of whichever library `foo.o` was linked into.

The promotion step is handled by `LinkUnitResolution`, a class whose interface is exposed only as a `friend` declaration to `NestedBuildNamespace` and `EntityName`. This is the correct seam for the link-time aggregation pass to insert itself without exposing namespace internals to checker authors.

### 47b.3.3 EntityId and EntityIdTable: Efficient Interning

`EntityName` uses string comparison for identity, which is correct but slow for large programs. `EntityIdTable` interns `EntityName` objects into dense integer-indexed handles:

```cpp
// clang/Analysis/Scalable/Model/EntityIdTable.h
namespace clang::ssaf {

class EntityId {
  size_t Index; // opaque; meaningful only within the same EntityIdTable
  explicit EntityId(size_t Index) : Index(Index) {}
  friend class EntityIdTable;
public:
  bool operator==(const EntityId &Other) const { return Index == Other.Index; }
  bool operator<(const EntityId &Other) const { return Index < Other.Index; }
};

class EntityIdTable {
  std::map<EntityName, EntityId> Entities;
public:
  EntityId getId(const EntityName &Name); // idempotent intern
  bool contains(const EntityName &Name) const;
  void forEach(llvm::function_ref<void(const EntityName &, EntityId)>) const;
  size_t count() const { return Entities.size(); }
};

} // namespace clang::ssaf
```

Checker logic during the link-time analysis step works with `EntityId` values — integer comparison is O(1) versus O(n) string comparison. The `EntityIdTable` is rebuilt from deserialized `EntityName` data at the start of each link-time invocation. For a program with 50,000 functions, the intern table occupies roughly 6–8 MB (assuming 120-byte average USR length), which is negligible relative to the summary corpus.

### 47b.3.4 SummaryName: Keying Analysis Results

Multiple different analyses may run on the same program entity — null-return tracking, ownership tracking, escape analysis — each producing a separate summary. `SummaryName` is the key under which a particular analysis result is stored:

```cpp
// clang/Analysis/Scalable/Model/SummaryName.h
namespace clang::ssaf {

class SummaryName {
  std::string Name; // e.g., "ssaf.null-return", "ssaf.ownership"
public:
  explicit SummaryName(std::string Name) : Name(std::move(Name)) {}
  bool operator==(const SummaryName &Other) const { return Name == Other.Name; }
  bool operator<(const SummaryName &Other) const { return Name < Other.Name; }
  llvm::StringRef str() const { return Name; }
};

} // namespace clang::ssaf
```

The design separates summary identity (`EntityName` identifies *what* entity) from analysis identity (`SummaryName` identifies *which property* of that entity). A single function thus maps to a set of `(SummaryName, value)` pairs, and the link-time checker selects only the summaries it cares about by filtering on `SummaryName`.

### 47b.3.5 Mapping AST Declarations to Entities

The bridge between Clang's AST and the SSAF data model is `ASTEntityMapping.h`:

```cpp
// clang/Analysis/Scalable/ASTEntityMapping.h
namespace clang::ssaf {

// Supported: functions, methods, global variables, parameters,
//            struct/class/union definitions, struct fields.
// NOT supported: implicit declarations, compiler builtins.
std::optional<EntityName> getEntityName(const Decl *D);

// Names the return value entity of a function.
std::optional<EntityName> getEntityNameForReturn(const FunctionDecl *FD);

} // namespace clang::ssaf
```

`getEntityName` calls into `clang::index::generateUSRForDecl` internally, appending the appropriate suffix and `NestedBuildNamespace`. The `std::optional` return handles the exclusion cases: compiler-synthesized declarations (copy constructors generated without source locations, `__builtin_*` functions) do not participate in cross-TU analysis because their behavior is modeled directly by the checker runtime.

The USR format used as the identity substrate is Clang's standard Unified Symbol Reference format, documented in `clang/include/clang/Index/USRGeneration.h`. For a free function `void foo(int)` in namespace `bar`, the USR is `c:@N@bar@F@foo#I#`. For a member function `void Widget::reset()`, it is `c:@S@Widget@F@reset#`. These strings are stable across re-compilations of the same TU — they depend only on the declaration's structure, not on object file layout or debug info.

The `Suffix` field in `EntityName` extends the USR with sub-entity discriminators. The currently defined suffixes are:
- `""` (empty): the declaration itself (a function's identity as a call target)
- `"return"`: the function's return value entity
- `"arg0"`, `"arg1"`, ...: individual parameter entities for ownership and escape tracking

This suffix scheme allows a single `FunctionDecl`'s USR to spawn multiple `EntityName` values without ambiguity, and each can carry independent summary facts — the function itself has a call graph edge fact, its return entity has a null-return fact, and its argument entities have ownership/escape facts.

`getEntityNameForReturn` returns a distinct `EntityName` for the return value of a function, enabling analyses that track "this function sometimes returns null" as a first-class fact attached to an entity, not as a side channel in the function's summary blob.

---

## 47b.4 Summary Format and Serialization

### 47b.4.1 Per-Function Summary Data Model

Each function in a TU produces a summary record containing a set of typed facts keyed by `SummaryName`. The logical structure — not yet exposed as a public header in 22.1.x — follows this schema:

```
FunctionSummary {
  EntityName    function;           // stable identity of this function
  EntityName?   returnEntity;       // identity of the return value (if non-void)

  // Callgraph edges
  CallEdge[] calls = [
    { EntityName callee; ArgBinding[] argBindings; }
  ];

  // Return value properties (keyed by SummaryName "ssaf.return-null")
  ReturnNullProperty? returnsNull;  // may-return-null, never-return-null, unknown

  // Argument escape/modification properties
  ArgEffect[] argEffects = [
    { unsigned argIndex; EffectKind kind; } // kinds: Freed, Escaped, Modified
  ];
}
```

The design is deliberately conservative: the per-TU pass does not attempt interprocedural reasoning — that is the job of the link-time step. Each `FunctionSummary` records only facts observable from the function's own body and its outgoing call edges.

### 47b.4.2 JSON Serialization Architecture

The serialization layer is accessed through `SerializationFormat` and `JSONWriter`, both of which are forward-declared as `friend` classes in `EntityName` and `NestedBuildNamespace`. This deliberate encapsulation ensures that the serialization format can evolve — switching from JSON to a compact binary format, for example — without breaking checker code that works only with the in-memory data model.

At the file level, a summary file is a JSON array of function summary objects:

```json
[
  {
    "entity": {
      "usr": "c:@F@allocate_buffer#I#",
      "suffix": "return",
      "namespace": [
        { "kind": "CompilationUnit", "name": "src/allocator.cpp" }
      ]
    },
    "summaries": {
      "ssaf.return-null": { "mayReturnNull": false },
      "ssaf.ownership": { "kind": "Owned", "escapesVia": [] }
    }
  },
  {
    "entity": {
      "usr": "c:@F@free_buffer#*v#",
      "suffix": "",
      "namespace": [
        { "kind": "CompilationUnit", "name": "src/allocator.cpp" }
      ]
    },
    "summaries": {
      "ssaf.ownership": { "kind": "Releases", "argIndex": 0 }
    }
  }
]
```

The USR strings in the JSON are the canonical cross-TU identifiers. The link-time aggregator reads all `.ssaf.json` files, calls `EntityIdTable::getId` on each deserialized `EntityName` to intern them into a shared table, and then operates exclusively on `EntityId` values for performance.

### 47b.4.3 Incremental Analysis and Caching

Because `.ssaf.json` files are generated per-TU at compile time, they participate in standard build-system dependency tracking. A TU whose source file has not changed since the last build does not need its summary regenerated; the link-time step can read the cached `.ssaf.json` from the previous build. This mirrors ThinLTO's use of LTO cache directories (see [Chapter 77 — LTO and ThinLTO](../part-13-lto-whole-program/ch77-lto-and-thinlto.md) §77.4).

The correctness requirement is that a summary's validity is tied to the TU's compilation fingerprint (source files, compile flags, macro definitions). Build systems that track these dependencies correctly — CMake's `depfile` mechanism, Bazel's hermetic sandbox, Ninja's dependency files — get incremental correctness for free, since the same conditions that invalidate a `.o` file also invalidate its `.ssaf.json`.

One subtlety is that the link-time analysis result depends on all summaries collectively, not just one. If `allocator.cpp` changes, the SSAF linker must re-run with the updated `allocator.ssaf.json` even if `client.cpp` and `registry.cpp` have not changed. The standard build-system dependency graph (the SSAF linker's output depends on all summary file inputs) handles this correctly: only the linker step re-runs, not the per-TU analyses for unchanged files. This property — that incremental cost is proportional to the number of changed TUs, not the size of the program — is the key scalability advantage of the SSAF approach over CTU analysis, which requires re-analyzing all TUs that can reach a changed callee through the call graph.

---

## 47b.5 Per-TU Analysis Phase

### 47b.5.1 What the Per-TU Pass Analyzes

The per-TU analysis pass is a Clang `FrontendAction` that runs after semantic analysis but before or alongside code generation. For each `FunctionDecl` in the TU, it:

1. Calls `getEntityName(FD)` to obtain the stable identifier.
2. Calls `getEntityNameForReturn(FD)` if the function is non-void.
3. Traverses the function body's AST to collect:
   - Direct call expressions → recorded as call edges with argument bindings
   - Return statements → return value properties (always-null, never-null)
   - Free/delete operations on parameters → `Freed` effect on the relevant argument index
   - Store-to-global / return-of-parameter-address → `Escaped` effect

The pass explicitly does **not** perform path-sensitive analysis. It computes may-properties (conservative approximations), not must-properties. A function with two return paths — one returning null, one returning a valid pointer — is summarized as `mayReturnNull: true`, which is sound but less precise than what the path-sensitive analyzer would report within a single TU.

Example invocation (conceptual; the flag name reflects SSAF's design direction and may differ from the final production flag):

```bash
# Compile and emit summary alongside the object file
clang-22 -c src/allocator.cpp -o build/allocator.o \
  -Xclang -analyze-ssaf \
  -Xclang -ssaf-output=build/allocator.ssaf.json \
  -Xclang -ssaf-compilation-id=src/allocator.cpp
```

The `-ssaf-compilation-id` flag sets the `CompilationUnit` namespace label baked into all `EntityName` entries emitted for this TU, enabling the link-time aggregator to disambiguate entities from different TUs with coincidentally identical USRs (a situation that arises with anonymous namespace functions that Clang assigns synthetic USRs).

### 47b.5.2 Integration with Compilation Databases

For projects using `compile_commands.json`, a driver script (`clang-ssaf-analyzer-driver`) wraps each compile command to append the SSAF flags and output path. The driver reads the compilation database, remaps output paths from `.o` to `.ssaf.json`, and invokes Clang with the augmented command line. This is the same pattern used by `analyze-build` (LLVM's `--ctu` mode) and by `run-clang-tidy`.

```bash
# Run SSAF per-TU analysis across all compile commands
clang-ssaf-analyzer-driver \
  --compilation-database=build/compile_commands.json \
  --output-dir=build/ssaf/ \
  --jobs=$(nproc)
```

The `--jobs` flag controls the number of parallel Clang invocations; since each TU is independent, full CPU parallelism is safe.

### 47b.5.3 Boundary with the Path-Sensitive Analyzer

SSAF and clang's path-sensitive static analyzer serve complementary roles and can run simultaneously without interference:

- The **path-sensitive analyzer** (`clang --analyze`) performs deep intra-procedural analysis. It tracks which specific path leads to a null dereference within a single function, generates the full bug path note sequence, and produces high-precision diagnostics. Its cross-TU extension (`-analyzer-config experimental-enable-naive-ctu-analysis=true` via `analyze-build --ctu`) works by importing foreign ASTs on demand — precise but slow.

- **SSAF** performs shallow inter-procedural analysis over compact summaries. It cannot tell you which path within `allocate_buffer` produces a null return; it can tell you that `allocate_buffer` *may* return null and that the callee in `client.cpp` does not check the return value before passing it to `use_buffer`. The resulting diagnostic identifies the cross-module data flow without enumerating the intra-module path.

The practical deployment pattern is to run both: the path-sensitive analyzer catches intra-TU bugs with high precision, and SSAF catches cross-TU bugs that the path-sensitive analyzer's analysis budget cannot reach.

A future integration direction is a **feedback loop** between the two analyses: SSAF summaries for a callee's return value can seed the path-sensitive analyzer's pre-analysis modeling for callers, reducing the number of unknown-state paths the analyzer must explore. For example, if SSAF knows that `get_handle()` never returns null (because its per-TU summary says `mayReturnNull: false`), the path-sensitive analyzer running on the caller can prune the null-return branch immediately without adding it to the worklist. This feedback reduces the path-sensitive analyzer's state space and its false-positive rate on null-dereference bugs — a direct payoff from the summary generation work done during compilation.

---

## 47b.6 Whole-Program Linking Phase

### 47b.6.1 Bottom-Up Callgraph Traversal

The link-time analysis pass (`clang-ssaf-linker`) reads all `.ssaf.json` summary files, constructs a whole-program callgraph from the recorded call edges, and traverses it in **bottom-up order** (leaves first, callers last). This traversal order is essential for interprocedural soundness: by the time the linker processes a call site in function `F`, it has already computed the complete summary for every callee of `F` that has a summary file.

```bash
# Run the link-time analysis step
clang-ssaf-linker \
  --summaries=build/ssaf/ \
  --link-unit=libfoo.so \
  --checkers=use-after-free,null-dereference,memory-leak \
  --output=build/ssaf-report.sarif
```

The `--link-unit` flag sets the `LinkUnit` namespace label, matching the `BuildNamespaceKind::LinkUnit` entry that the linker promotes each entity's `NestedBuildNamespace` to. After promotion, every function's `EntityName` carries a two-level `NestedBuildNamespace`: first the `CompilationUnit` namespace identifying which `.cpp` file it came from, then the `LinkUnit` namespace identifying which library it was linked into. This two-level qualification is what allows the checker to produce diagnostics that say "allocated in `src/allocator.cpp` (part of `libfoo.so`) and freed by a function from `src/client.cpp` (part of `app`)."

The linker's entry point iterates the callgraph's strongly connected components (SCCs) in reverse topological order, using Tarjan's algorithm or Kosaraju's algorithm to compute the SCC decomposition. Within an SCC (a set of mutually recursive functions), it applies a fixed-point iteration until the summaries stabilize — the same worklist algorithm used by dataflow analyses in `llvm/lib/Analysis/`. In practice, SCCs are small (the majority are single functions), so fixed-point convergence is fast. Real-world C++ codebases rarely have large mutual recursion cycles at the library boundary level; the pathological case is a deeply recursive graph where each SCC iteration changes one fact. The linker bounds this with a configurable maximum iteration count (default: 10) and conservatively marks non-converged summaries as "unknown" to avoid infinite loops.

Functions that lack `.ssaf.json` summaries — external library functions not compiled with SSAF — are treated as **opaque callees**: their effects are modeled conservatively. A call to an opaque function that receives a pointer argument is assumed to potentially free that pointer, store it to a global (marking it Escaped), and return null. Checkers can register per-callee models that override this default for well-known functions (`malloc`, `free`, POSIX APIs) using a model database analogous to the clang static analyzer's `StdLibraryFunctionsChecker` modeling infrastructure.

### 47b.6.2 The Checker Transfer Function Interface

Each SSAF checker implements a transfer function that describes how a call site affects the caller's summary state given the callee's summary. The logical interface is:

```cpp
namespace clang::ssaf {

// Conceptual interface for link-time SSAF checkers.
// Exposed via the checker plugin API, not as a public shipped header in 22.1.x.
class SsafCheckerBase {
public:
  virtual ~SsafCheckerBase() = default;

  // Called once per function during the bottom-up traversal.
  // Summarize the function body given the already-computed summaries
  // of all callees. Returns the summary facts to attach to this function.
  virtual FunctionSummaryFacts
  summarizeFunction(const EntityName &FN,
                    const CallGraphNode &Node,
                    const SummaryStore &CalleeStore) = 0;

  // Called when the traversal finds a potentially buggy pattern at a
  // call site. Emits a diagnostic if the pattern is confirmed.
  virtual void
  visitCallSite(const CallSite &CS,
                const FunctionSummaryFacts &CalleeFacts,
                DiagnosticEmitter &Emitter) = 0;

  // Returns the SummaryName this checker produces and consumes.
  virtual SummaryName checkerSummaryName() const = 0;
};

} // namespace clang::ssaf
```

The `FunctionSummaryFacts` type is a map from `SummaryName` to opaque value blobs — checkers read their own entries and ignore others. The `SummaryStore` passed to `summarizeFunction` is keyed by `EntityName` (or by the interned `EntityId` for performance), enabling O(1) lookup of any callee's facts.

### 47b.6.3 Built-in Cross-TU Checkers

The SSAF framework is designed to ship with three initial built-in checkers for cross-TU bug classes that the path-sensitive analyzer cannot reach:

**UseAfterFree (cross-TU)**: Tracks heap objects that escape a module boundary (allocated in one TU, stored in a struct or returned through an API, then freed by a function in a third TU). The per-TU pass records `EffectKind::Freed` on argument 0 of `free_buffer`; the per-TU pass for the client TU records that the argument to `free_buffer` derives from a stored pointer. The link-time step composes these facts: the stored pointer is freed, and any subsequent use of the struct field is a use-after-free.

**NullDereference (cross-TU)**: Propagates `ssaf.return-null: mayReturnNull: true` facts from callees to callers. If library function `get_handle()` may return null and the caller in another TU dereferences the return value without checking, the link-time checker flags a cross-TU null dereference — a bug invisible to per-TU analysis because `get_handle`'s body is in a different TU.

**MemoryLeak (cross-TU)**: Identifies allocations whose `Owned` summary fact is never matched by a `Releases` fact anywhere in the whole-program callgraph. The checker accumulates all `ssaf.ownership: Owned` facts, then walks the callgraph looking for a `Releases` edge that consumes each allocation. Unmatched allocations that are reachable from program entry and have no `Escaped` tag are reported as leaks.

### 47b.6.4 False Positive Rate Tradeoffs

Summary-based analysis trades precision for scalability. The key conservatisms that introduce false positives are:

- **May-analysis**: `mayReturnNull: true` is recorded if *any* path returns null, even if the callee is always invoked in a context where that path is unreachable.
- **Missing context sensitivity**: The summary for a polymorphic function (one taking a function pointer or virtual dispatch) conservatively assumes the worst-case callee behavior unless the call graph is resolved precisely.
- **Escaped fact suppression**: A pointer marked `Escaped` (stored to a global or returned by address) cannot have its lifetime tracked across the escape boundary, so ownership facts are lost. The checker suppresses ownership diagnostics for escaped pointers to avoid false positives, potentially missing real leaks.

In practice, experienced teams run SSAF with a suppression file that whitelists known-safe patterns (e.g., "objects allocated in `init()` and freed in `teardown()` are always matched") before triaging the raw output. A SARIF-format suppression file lists `suppressions` entries with USR-based fingerprints that match the `EntityName` of the suppressed finding, allowing the suppression set to survive minor refactors that change source locations without changing the USR of the involved functions.

---

## 47b.7 Writing an SSAF Checker

### 47b.7.1 The Checker Extension Points

An SSAF checker is a plugin shared library that registers itself with the linker's checker manager. The registration pattern mirrors the clang static analyzer's CRTP checker registration (see [Chapter 45 — The Static Analyzer](ch45-the-static-analyzer.md) §45.5.1), adapted for the link-time context:

```cpp
// MyNullChecker.cpp
#include "clang/Analysis/Scalable/Model/SummaryName.h"
#include "clang/Analysis/Scalable/Model/EntityName.h"
// Link-time checker API (emerges from SsafCheckerBase):
#include "clang/Analysis/Scalable/Checker/SsafCheckerBase.h"

namespace {

class CrossTUNullChecker : public clang::ssaf::SsafCheckerBase {
public:
  clang::ssaf::SummaryName checkerSummaryName() const override {
    return clang::ssaf::SummaryName("ssaf.null-return");
  }

  // ... (see §47b.7.2)
};

} // namespace

// Registration entry point called by the checker plugin loader.
extern "C" void registerSsafCheckers(clang::ssaf::CheckerRegistry &R) {
  R.addChecker<CrossTUNullChecker>("CrossTU.NullReturn",
                                   "Cross-TU null return propagation");
}
```

### 47b.7.2 Example: Cross-TU Null Propagation Checker

The following checker demonstrates the key lifecycle callbacks. It propagates `mayReturnNull` facts bottom-up through the callgraph and reports dereferences of potentially-null return values at call sites in the caller TU.

```cpp
clang::ssaf::FunctionSummaryFacts
CrossTUNullChecker::summarizeFunction(
    const clang::ssaf::EntityName &FN,
    const clang::ssaf::CallGraphNode &Node,
    const clang::ssaf::SummaryStore &CalleeStore) {

  clang::ssaf::FunctionSummaryFacts Facts;
  NullReturnFact Fact;

  // Check if this function's own body may return null.
  // (populated by the per-TU pass from AST inspection)
  if (auto *OwnFact = CalleeStore.lookupOwn(FN, checkerSummaryName()))
    Fact.mayReturnNull |= OwnFact->mayReturnNull;

  // Propagate: if this function calls a callee that may return null
  // and passes the return value through without a null check,
  // then this function also may return null.
  for (const auto &Edge : Node.outgoingEdges()) {
    if (auto *CalleeFact =
            CalleeStore.lookup(Edge.callee(), checkerSummaryName())) {
      if (CalleeFact->mayReturnNull && !Edge.hasNullCheckOnReturn())
        Fact.mayReturnNull = true;
    }
  }

  Facts.set(checkerSummaryName(), Fact);
  return Facts;
}

void CrossTUNullChecker::visitCallSite(
    const clang::ssaf::CallSite &CS,
    const clang::ssaf::FunctionSummaryFacts &CalleeFacts,
    clang::ssaf::DiagnosticEmitter &Emitter) {

  auto *Fact = CalleeFacts.get<NullReturnFact>(checkerSummaryName());
  if (!Fact || !Fact->mayReturnNull)
    return;

  // The callee may return null. Check if the return value is
  // dereferenced at the call site without an intervening null check.
  if (CS.returnValueDereferencedWithoutCheck()) {
    Emitter.emit(
        CS.location(),
        "Cross-TU null dereference: callee '" +
            CS.calleeName() + "' may return null "
            "(defined in " + CS.calleeTUName() + ")",
        DiagnosticSeverity::Warning);
  }
}
```

The `NullReturnFact` struct is a plain-old-data type serialized by the framework's `SerializationFormat` using standard LLVM JSON support:

```cpp
struct NullReturnFact {
  bool mayReturnNull = false;

  // Called by SerializationFormat for JSON emission.
  llvm::json::Value toJSON() const {
    return llvm::json::Object{{"mayReturnNull", mayReturnNull}};
  }

  static std::optional<NullReturnFact> fromJSON(const llvm::json::Value &V) {
    NullReturnFact F;
    if (auto *Obj = V.getAsObject())
      if (auto MRN = Obj->getBoolean("mayReturnNull"))
        F.mayReturnNull = *MRN;
    return F;
  }
};
```

### 47b.7.3 Linking the Checker into the Build

SSAF checker plugins are shared libraries registered with the linker via `--load-plugin`:

```bash
clang-ssaf-linker \
  --summaries=build/ssaf/ \
  --link-unit=myapp \
  --load-plugin=build/lib/CrossTUNullChecker.so \
  --output=build/ssaf-report.sarif
```

The CMake target for a checker plugin:

```cmake
add_library(CrossTUNullChecker MODULE
  CrossTUNullChecker.cpp
)
target_link_libraries(CrossTUNullChecker PRIVATE
  clangAnalysisScalable   # EntityName, SummaryName, EntityIdTable
  LLVMSupport             # llvm::json, llvm::StringRef, etc.
)
set_target_properties(CrossTUNullChecker PROPERTIES
  PREFIX ""   # omit "lib" prefix so loader finds "CrossTUNullChecker.so"
)
```

---

## 47b.8 Integration with Build Systems

### 47b.8.1 CMake Integration

The per-TU analysis pass runs as a custom command alongside each object file compilation. The `add_custom_command` pattern wraps the standard `clang` invocation:

```cmake
function(add_ssaf_analysis target)
  get_target_property(SOURCES ${target} SOURCES)
  set(SSAF_OUTPUTS "")

  foreach(SRC ${SOURCES})
    get_filename_component(BASE ${SRC} NAME_WE)
    set(OBJ  "${CMAKE_CURRENT_BINARY_DIR}/${BASE}.o")
    set(SSAF "${CMAKE_CURRENT_BINARY_DIR}/ssaf/${BASE}.ssaf.json")
    list(APPEND SSAF_OUTPUTS ${SSAF})

    add_custom_command(
      OUTPUT ${OBJ} ${SSAF}
      COMMAND clang-22 -c ${SRC}
              -o ${OBJ}
              -Xclang -analyze-ssaf
              -Xclang -ssaf-output=${SSAF}
              -Xclang -ssaf-compilation-id=${SRC}
              # Forward the same compile flags as the real build:
              "$<TARGET_PROPERTY:${target},COMPILE_OPTIONS>"
      DEPENDS ${SRC}
      COMMENT "Compiling and analyzing ${SRC}"
    )
  endforeach()

  # Phony target that depends on all summary files.
  add_custom_target(${target}_ssaf_link
    COMMAND clang-ssaf-linker
            --summaries=${CMAKE_CURRENT_BINARY_DIR}/ssaf/
            --link-unit=$<TARGET_FILE_NAME:${target}>
            --checkers=use-after-free,null-dereference,memory-leak
            --output=${CMAKE_CURRENT_BINARY_DIR}/${target}-ssaf-report.sarif
    DEPENDS ${SSAF_OUTPUTS}
    COMMENT "Running whole-program SSAF analysis for ${target}"
  )
endfunction()
```

### 47b.8.2 Ninja Phony Targets

In Ninja-generated builds, the dependency graph is explicit. The SSAF linker target depends on all `.ssaf.json` files, which in turn depend on their source files — identical structure to `.o` file dependencies:

```
# build.ninja (generated excerpt)
rule ssaf_compile
  command = clang-22 -c $in -o $out.o \
            -Xclang -analyze-ssaf \
            -Xclang -ssaf-output=$out.ssaf.json
  description = Compile+Analyze $in

build allocator.o allocator.ssaf.json: ssaf_compile src/allocator.cpp
build client.o    client.ssaf.json:    ssaf_compile src/client.cpp

rule ssaf_link
  command = clang-ssaf-linker \
            --summaries=ssaf/ \
            --link-unit=libfoo.so \
            --output=ssaf-report.sarif
  description = SSAF whole-program analysis

build ssaf-report.sarif: ssaf_link allocator.ssaf.json client.ssaf.json
```

Ninja's dependency tracking ensures that if `src/allocator.cpp` changes, `allocator.ssaf.json` is regenerated but `client.ssaf.json` is not, and the SSAF linker reruns using the updated allocator summary and the cached client summary.

### 47b.8.3 CI Parallelism Pattern

The canonical CI pipeline for SSAF-enabled projects runs in three phases:

```
Phase 1 (parallel): Compile all TUs, emit .o and .ssaf.json
  ├─ clang-22 -c foo.cpp -o foo.o -Xclang -analyze-ssaf ...
  ├─ clang-22 -c bar.cpp -o bar.o -Xclang -analyze-ssaf ...
  └─ clang-22 -c baz.cpp -o baz.o -Xclang -analyze-ssaf ...

Phase 2 (sequential): Link the binary
  └─ lld foo.o bar.o baz.o -o libfoo.so

Phase 3 (sequential): Run SSAF whole-program analysis
  └─ clang-ssaf-linker --summaries=ssaf/ --link-unit=libfoo.so
                       --output=ssaf-report.sarif
```

Phase 1 runs with the same parallelism as a normal build (`make -j$(nproc)`). Phase 3 is typically fast relative to compilation because it processes only summary files, not source ASTs. For a project with 5,000 TUs and 50,000 exported functions, the link-time step typically runs in under 30 seconds on a 16-core machine, compared to several minutes for a full CTU analysis with clang's AST import mechanism.

### 47b.8.4 Bazel Integration

In Bazel's hermetic execution model, every action's inputs and outputs are declared up front. The SSAF per-TU pass fits naturally as a second output alongside the object file:

```python
# tools/ssaf/ssaf.bzl
def ssaf_cc_library(name, srcs, hdrs=[], deps=[], **kwargs):
    """cc_library wrapper that also produces .ssaf.json summary files."""
    ssaf_outs = [src.replace(".cpp", ".ssaf.json") for src in srcs]
    ssaf_dir  = name + "_ssaf_summaries"

    # Run per-TU analysis for each source file.
    for src in srcs:
        ssaf_out = src.replace(".cpp", ".ssaf.json")
        native.genrule(
            name = src.replace(".cpp", "_ssaf"),
            srcs = [src] + hdrs,
            outs = [ssaf_out],
            cmd  = ("$(location //tools/ssaf:clang_ssaf_wrapper) "
                    "--source=$(location {src}) "
                    "--output=$@ "
                    "--compilation-id={src}").format(src=src),
            tools = ["//tools/ssaf:clang_ssaf_wrapper"],
        )

    # Aggregate all summary files as a filegroup.
    native.filegroup(
        name = ssaf_dir,
        srcs = ssaf_outs,
    )

    # Standard cc_library for the actual build.
    native.cc_library(
        name = name,
        srcs = srcs,
        hdrs = hdrs,
        deps = deps,
        **kwargs
    )
```

The Bazel link-time analysis target then takes all summary filegroups as inputs:

```python
# BUILD file
ssaf_whole_program_analysis(
    name = "myapp_ssaf",
    summaries = [
        ":allocator_ssaf_summaries",
        ":registry_ssaf_summaries",
        ":client_ssaf_summaries",
    ],
    link_unit = "myapp",
    checkers = ["use-after-free", "null-dereference", "memory-leak"],
    output = "myapp-ssaf-report.sarif",
)
```

Because Bazel tracks content hashes rather than timestamps, the SSAF link step re-runs only when a summary file's content actually changes — not whenever its source file is touched — providing tighter incrementality than make-based systems.

---

## 47b.9 Comparison with Other Whole-Program Approaches

| Approach | Scope | Scalability | Precision | Cross-TU |
|---|---|---|---|---|
| clang static analyzer (per-TU) | Single TU | Medium (function-budget limited) | High (path-sensitive) | No |
| clang CTU via AST import | Whole program | Low–Medium (AST re-parsing) | High (path-sensitive) | Yes |
| SSAF (summary-based) | Whole program | High (ThinLTO-scale) | Medium (summary-approximate) | Yes |
| CodeChecker + CTU | Whole program | Low–Medium | High (symbolic) | Yes |
| Infer (Facebook/Meta) | Whole program | High | Medium (biabduction) | Yes |
| clang-tidy | Per-TU | High | Low (AST-pattern only) | No |

**clang CTU via AST import** (`-analyzer-config experimental-enable-naive-ctu-analysis=true`) achieves full path-sensitive precision across TU boundaries by importing foreign ASTs on demand during analysis. The cost is that each cross-TU call potentially triggers re-parsing a foreign TU and re-running its analyses; for large projects, this multiplies analysis time by 10x or more compared to per-TU analysis. The preparation step — running `clang-extdef-mapping` to generate the USR-to-AST-file index — requires a full project compilation with `-emit-ast` before analysis can begin, adding an additional build phase. In contrast, SSAF summary generation runs inside the same compilation that produces object files, adding no extra build phases.

**Infer** uses biabduction — a form of separation logic inference — to compute pre/postcondition summaries that are more powerful than SSAF's may-properties but require the Infer analysis engine as an external dependency. Infer's summaries are sound for memory safety (null dereferences, use-after-free, resource leaks) but its semantics differ from Clang's, making it unsuitable for checker authors who want to reuse Clang's AST matchers and static analyzer infrastructure.

**CodeChecker** is a server-based platform that wraps clang's static analyzer (including CTU mode) with a web UI for triaging results, suppressing false positives, and tracking regressions. CodeChecker can consume SARIF output from the SSAF linker alongside its own clang analyzer reports, making it a natural integration point for teams running both tools: per-TU path-sensitive bugs appear alongside cross-TU SSAF bugs in a unified review interface.

**SSAF's** advantage is **integration**: checker authors write C++ against LLVM APIs, the analysis pipeline is invokable from standard build systems, and the diagnostic output uses SARIF — the same format as clang's static analyzer. Teams already using clang as their compiler can add SSAF analysis without introducing an external tool chain dependency.

**Precision versus coverage** is the key axis along which these tools differ. Path-sensitive per-TU analysis misses no intra-TU bugs on the analyzed paths but is blind to cross-module data flow. SSAF may produce false positives from conservative may-analysis but covers bugs that are structurally impossible to detect per-TU. Operationally, most production security teams run path-sensitive analysis as the gating check (zero-tolerance for confirmed bugs) and SSAF as the advisory check (triage required), suppressing known-false-positive patterns via a checked-in SSAF suppressions file.

---

## 47b.10 Practical Example: Cross-Module Use-After-Free

### 47b.10.1 The Scenario

Consider a three-file program:

```cpp
// allocator.cpp  (compiled into libfoo.so)
Buffer *allocate_buffer(size_t sz) {
  return new Buffer(sz);  // always non-null; freed by free_buffer
}

void free_buffer(Buffer *buf) {
  delete buf;
}
```

```cpp
// registry.cpp  (compiled into libfoo.so)
static Buffer *g_registry[MAX_ENTRIES];

void register_buffer(int id, Buffer *buf) {
  g_registry[id] = buf;  // buf escapes to global
}

Buffer *get_buffer(int id) {
  return g_registry[id];
}
```

```cpp
// client.cpp  (compiled into the application binary)
#include "allocator.h"
#include "registry.h"

void process() {
  Buffer *b = allocate_buffer(4096);
  register_buffer(0, b);
  free_buffer(b);                // BUG: b is freed here...
  Buffer *r = get_buffer(0);
  r->write("hello");             // ...but used here via the registry
}
```

The path-sensitive analyzer running on `client.cpp` alone cannot detect this bug: `free_buffer` is opaque (defined in a separate TU), so the analyzer does not know it frees its argument. Similarly, `register_buffer` is opaque, so the analyzer does not know it stores the pointer into a global. The bug is invisible to per-TU analysis.

### 47b.10.2 How Summary Facts Propagate

After per-TU SSAF analysis:

**`allocator.ssaf.json`** records:
- `allocate_buffer` return entity: `ssaf.ownership: { kind: Owned }` (new allocation)
- `free_buffer` argument 0: `ssaf.ownership: { kind: Releases, argIndex: 0 }`

**`registry.ssaf.json`** records:
- `register_buffer` argument 1: `ssaf.ownership: { kind: Escaped, target: Global }`
- `get_buffer` return entity: `ssaf.ownership: { kind: BorrowedFromGlobal }`

**`client.ssaf.json`** records (using stable `EntityName` references):
- `process` calls: `allocate_buffer` → local `b`, `register_buffer(0, b)`, `free_buffer(b)`, `get_buffer(0)` → local `r`, `r->write(...)`

The link-time UseAfterFree checker performs the following reasoning during bottom-up traversal:

1. `free_buffer` summary: argument 0 is `Releases` — any `EntityId` passed as argument 0 to `free_buffer` is considered freed.
2. `register_buffer` summary: argument 1 is `Escaped` to a global — the `EntityId` passed as argument 1 is reachable from global state.
3. In `process`: `b` is passed to both `register_buffer(0, b)` (escapes to global) and `free_buffer(b)` (freed). The `EntityId` for `b` acquires both `Freed` and `EscapedToGlobal` facts.
4. `get_buffer` returns a `BorrowedFromGlobal` — the `EntityId` returned may alias any globally escaped `Buffer *`.
5. The checker detects: `b` is freed, `b` is globally escaped, and `get_buffer` returns a value that may alias `b`. A call to `r->write()` on a value aliasing a freed pointer is a use-after-free.

### 47b.10.3 Producing the Diagnostic

The SSAF linker emits a SARIF diagnostic:

```json
{
  "version": "2.1.0",
  "runs": [{
    "tool": { "driver": { "name": "clang-ssaf-linker", "version": "22.1.6" }},
    "results": [{
      "ruleId": "CrossTU.UseAfterFree",
      "message": {
        "text": "Use of freed pointer: 'b' freed by 'free_buffer' (allocator.cpp:7), but reachable via 'get_buffer' (registry.cpp:11) through global state"
      },
      "locations": [{
        "physicalLocation": {
          "artifactLocation": { "uri": "client.cpp" },
          "region": { "startLine": 10 }
        }
      }],
      "relatedLocations": [
        { "message": { "text": "Freed here" },
          "physicalLocation": { "artifactLocation": { "uri": "client.cpp" },
                                "region": { "startLine": 8 }}},
        { "message": { "text": "Escaped to global here" },
          "physicalLocation": { "artifactLocation": { "uri": "client.cpp" },
                                "region": { "startLine": 7 }}}
      ]
    }]
  }]
}
```

The SARIF format is consumable by CodeChecker's web UI, GitHub's code scanning interface, and VS Code's SARIF viewer extension — the same tooling used for clang static analyzer output.

---

## 47b.11 Chapter Summary

- SSAF (Scalable Static Analysis Framework) solves the whole-program analysis problem by separating per-TU summary generation from link-time aggregation, using the same two-phase pattern as ThinLTO.

- LLVM 22.1.x ships the SSAF data model in `clang/Analysis/Scalable/` and `libclangAnalysisScalable.a`. The core types — `EntityName`, `EntityId`, `EntityIdTable`, `SummaryName`, `BuildNamespace`, `NestedBuildNamespace` — are stable, tested APIs in the `clang::ssaf` namespace.

- `EntityName` provides globally unique, cross-TU-stable identity for program entities, built on Clang's USR infrastructure and extended with a build-system namespace hierarchy (`CompilationUnit` → `LinkUnit`).

- `NestedBuildNamespace` models the ThinLTO-analogous CU-to-LU promotion: entities start as `CompilationUnit`-qualified and are promoted to `LinkUnit`-qualified by the link-time `LinkUnitResolution` layer.

- `EntityIdTable` interns `EntityName` values into dense integer handles for O(1) comparison during the link-time traversal — essential for programs with tens of thousands of analyzed functions.

- `getEntityName` and `getEntityNameForReturn` (in `ASTEntityMapping.h`) are the bridge from Clang's AST `Decl` nodes to the SSAF data model; they are the correct entry points for per-TU analysis passes.

- The link-time analysis performs a bottom-up callgraph traversal, propagating summary facts through call edges with a fixed-point iteration for recursive SCCs. This is O(program edges × checker complexity), not O(program paths).

- SSAF's per-TU summaries participate in standard build-system dependency tracking, enabling fully incremental whole-program analysis: only re-analyze TUs whose source has changed.

- Compared to full CTU analysis (AST import), SSAF trades per-finding precision for scalability. Compared to per-TU analysis, it provides cross-module coverage. The recommended deployment combines both: per-TU path-sensitive analysis for intra-module precision, SSAF for cross-module coverage.

- Custom checkers extend `SsafCheckerBase` and implement `summarizeFunction` (to propagate facts bottom-up) and `visitCallSite` (to detect buggy patterns), then register via a plugin `registerSsafCheckers` entry point.

- The SSAF data model's separation of entity identity (`EntityName` keyed by USR) from analysis identity (`SummaryName` keyed by checker name) allows multiple checkers to annotate the same entity with independent, non-interfering facts, and allows the serialization format to evolve from JSON to a compact binary representation without changing checker code.

- SSAF's long-term integration direction includes feeding summaries back into the per-TU path-sensitive analyzer to prune unreachable paths (e.g., a function known to never return null does not need its null-return branch explored in callers), reducing false positives while increasing the analyzer's effective depth.

- The `clang::ssaf` namespace, `libclangAnalysisScalable.a`, and all six shipped headers (`ASTEntityMapping.h`, `Model/BuildNamespace.h`, `Model/EntityId.h`, `Model/EntityIdTable.h`, `Model/EntityName.h`, `Model/SummaryName.h`) are the stable extension surface in LLVM 22.1.x; the per-TU emission pass and link-time binary are the active development frontier as of mid-2026.
