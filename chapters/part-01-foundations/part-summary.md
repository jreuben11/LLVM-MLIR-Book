# Part I — Foundations — Part Summary

*This part establishes the structural, operational, and programmatic bedrock of the entire LLVM ecosystem, equipping the reader to navigate the monorepo, build LLVM from source, trace a program through every stage of the compilation pipeline, wield the C++ API idioms that permeate every subsequent chapter, and integrate LLVM components as reusable libraries.*

---

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 1 | The LLVM Project | History, monorepo layout, subprojects, versioning, and downstream ecosystem |
| 2 | Building LLVM from Source | CMake variables, build types, stage builds, acceleration, and cross-compilation |
| 3 | The Compilation Pipeline | Source-to-binary stages, tool roles, LTO, offload, MLIR, and ORC JIT |
| 4 | The LLVM C++ API | Custom RTTI, LLVMContext, ADT containers, Error handling, and IR iteration |
| 5 | LLVM as a Library | llvm-config, CMake integration, component model, C API, and pass plugins |

---

## Part Overview

Part I addresses the single obstacle that stops most compiler engineers before they write their first pass: LLVM is large, deliberately unstable across major versions, and built on design choices that look arbitrary until explained. These five chapters remove that obstacle systematically.

Chapter 1 establishes what LLVM actually is — not a compiler, not a language, not a runtime, but a federated toolkit of independently linkable libraries sharing a stable IR and a common pass infrastructure. It traces the design from Lattner's 2002 MS thesis through Apple's 2005 adoption, the 2019 monorepo migration, and the six-month release cadence formalized at LLVM 14. It maps every active subproject in the monorepo and surveys the major industrial downstreams (Apple, Android, Rust, Intel, AMD, NVIDIA) to explain why the library-first design philosophy has propagated so far beyond its original academic scope. After Chapter 1, the reader understands the terrain and can locate any component within it.

Chapter 2 turns understanding into action. Building from source is not optional for anyone who will modify passes, write plugins, or profile the compiler itself. The chapter systematically covers every CMake variable that matters — `LLVM_TARGETS_TO_BUILD`, `BUILD_SHARED_LIBS`, `LLVM_USE_SPLIT_DWARF`, `LLVM_PARALLEL_LINK_JOBS`, and the rest — explains the architectural distinction between projects (compiled with the host compiler) and runtimes (compiled with the just-built Clang), and walks through 1-stage, 2-stage, and 3-stage PGO-guided bootstrap builds. Build acceleration with ccache, sccache, distcc, and llvm-cas is covered in enough depth to get a developer build from 15 minutes to 30 seconds on a warm cache. Cross-compilation to AArch64 Linux and bare-metal ARM rounds out the chapter.

Chapter 3 maps the full compilation pipeline from source to linked binary, making every intermediate representation observable with a one-liner. It dissects the driver/cc1 split, shows how `-###` reveals the exact cc1 invocation, and walks through the seven pipeline stages — preprocessed source, AST, LLVM IR, SelectionDAG or generic MIR, MachineIR, MC, and object/linked binary — with concrete commands for inspecting each. The chapter extends this map to LTO (monolithic and ThinLTO), GPU offload (CUDA, HIP, OpenMP target), the MLIR lowering chain, Flang's FIR and HLFIR dialects, and ORC JIT lazy compilation. After Chapter 3, the reader knows which tool owns which stage and how to insert an observation point at any level.

Chapter 4 covers the C++ API idioms that every LLVM subsystem is built from. The custom `isa<>`/`cast<>`/`dyn_cast<>` system replaces `dynamic_cast` with opcode-range checks that inline at the call site. `LLVMContext` is the thread-local universe that owns all type and metadata state. The ADT library — `SmallVector`, `StringRef`, `Twine`, `DenseMap`, `SetVector`, `StringMap`, and their siblings — provides performance-tuned containers with specific lifetime invariants that the reader must internalize. `llvm::Error` and `llvm::Expected<T>` implement checked, exception-free error handling. `cl::opt<T>` provides global declarative option registration. Range-for over `Module → Function → BasicBlock → Instruction`, use-def traversal via `Value::uses()`, `GraphTraits`-based DFS/BFS, and `InstVisitor` CRTP dispatch complete the toolkit. These primitives are the vocabulary of every later chapter.

Chapter 5 closes the loop by explaining how to consume LLVM as a library from an external project. `llvm-config` provides compilation and link flags for quick builds; `find_package(LLVM REQUIRED CONFIG)` with `llvm_map_components_to_libnames()` is the correct CMake integration for production projects. The component model — approximately 100 named libraries with an explicit dependency graph — enables minimal linking: a bitcode transformer links `core`, `support`, `irreader`, and `bitwriter` without pulling in any target backend. The stable C API (`llvm-c/`) gives Python (llvmlite), Rust (llvm-sys), and other FFI consumers a stable ABI target that the C++ API cannot provide. The chapter closes with a complete, working out-of-tree pass plugin skeleton and a troubleshooting reference for the linking errors that trap every first-time integrator.

---

## Key Concepts Introduced

- **LLVM as a toolkit, not a compiler.** LLVM has no single binary called `llvm`; it is a collection of independently linkable libraries. `clang` is a front end, `opt` is an optimizer driver, `llc` is a code generator, and each is a thin shell around shared library components.

- **The monorepo and co-versioning.** All LLVM subprojects live at `github.com/llvm/llvm-project` and are tagged together (`llvmorg-NN.M.P`). A single commit can atomically span Clang, the optimizer, MLIR, LLD, and compiler-rt.

- **The six-month major release cadence.** Formalized at LLVM 14, the cadence produces two major releases per year. The C++ API has no ABI stability guarantee between releases; only the C API (`llvm-c/`) and `libclang` maintain stability.

- **Projects vs. runtimes.** `LLVM_ENABLE_PROJECTS` lists subprojects compiled with the host compiler; `LLVM_ENABLE_RUNTIMES` lists libraries compiled with the just-built Clang. The distinction is essential for bootstrap builds.

- **The compilation pipeline and its observable stages.** Source → preprocessed source → Clang AST → LLVM IR → SelectionDAG or generic MIR → MachineIR → MC → object → linked binary. Every stage is observable with a one-liner (`-emit-llvm -S`, `-stop-after=finalize-isel`, `llvm-objdump -d`, etc.).

- **Driver vs. cc1.** The `clang` binary is a driver that constructs a job graph and forks child processes. `clang -cc1` is the actual compiler. `clang -###` reveals the exact cc1 invocation and shows how driver flags (e.g., `-O2`) map to dozens of cc1 and LLVM flags.

- **ThinLTO and the module summary index.** ThinLTO stores a compact call-graph summary in each bitcode object. At link time, the linker reads only summaries to determine cross-unit import candidates, then compiles each module in parallel. This achieves 70–90% of monolithic LTO's speedup at a fraction of the link time.

- **ORC JIT and lazy compilation.** The ORC (On-Request Compilation) framework compiles IR or objects on-demand in-process, out-of-process, or remotely. Stubs defer function-body compilation until first call, enabling interactive REPLs and language runtimes with fast startup. Julia, Swift, and LLDB's expression evaluator all build on ORC.

- **Custom RTTI: `isa<>`, `cast<>`, `dyn_cast<>`.** LLVM's casting system replaces `dynamic_cast` with inlinable opcode-range checks via a static `classof` method on each class. The `isa_impl<To,From>` trait (LLVM 14+) enables non-intrusive specialization for external hierarchies.

- **`LLVMContext` and the one-context-per-thread rule.** All IR state — types, metadata, attribute sets, diagnostic handlers — is owned by an `LLVMContext`. The context is not thread-safe by design; each thread owns its own context and modules are merged explicitly via `llvm::Linker`.

- **ADT containers and their invariants.** `SmallVector<T,N>` avoids heap allocation for small counts. `StringRef` and `ArrayRef<T>` are non-owning views that must not outlive their backing storage. `Twine` is a lazy concatenation type that must never be stored. `DenseMap` invalidates all iterators and references on insertion. `SetVector` provides O(1) membership with deterministic insertion-order iteration.

- **`llvm::Error` and `llvm::Expected<T>`.** Checked, exception-free error propagation. Both are `[[nodiscard]]`; every error value must be consumed before destruction or the program terminates. Custom error types inherit from `ErrorInfo<Derived>` with a static `char ID` as the type tag.

- **The component model and minimal linking.** LLVM compiles to approximately 100 named static libraries with an explicit dependency graph. `llvm_map_components_to_libnames()` resolves component names to library targets and their transitive dependencies. Tools that do not lower to native code can omit all target backends, reducing binary size by an order of magnitude.

- **Out-of-tree pass plugins.** A shared library defining `llvmGetPassPluginInfo()` can register new passes with the `PassBuilder` at load time, without modifying the LLVM source tree. Loaded with `opt --load-pass-plugin=./MyPlugin.so --passes="my-pass"`. The plugin must not link against LLVM libraries — symbols are resolved from the host process.

---

## How This Part Fits the Book

This is the first part; nothing precedes it. The material here is prerequisite for every subsequent chapter: the IR chapters in Part IV assume Chapter 3's pipeline map and Chapter 4's API vocabulary; the Clang chapters in Parts V–VIII assume Chapter 1's subproject knowledge and Chapter 2's build competency; the pass-writing chapters in Part X assume Chapter 4's ADT library and Chapter 5's plugin infrastructure; the MLIR chapters in Parts XIX–XXIII assume Chapter 3's MLIR pipeline overview. Parts II and III (compiler theory and type theory) are largely independent of Part I but use the same pipeline framing. Parts XXIX–XXXI (tooling, AI-first PL design, and frontier evolution) build most directly on Chapters 1, 3, and 5 of this part.

---

## Cross-Part Dependencies

The following chapters and parts draw directly on material introduced here:

- **Ch 16–27 (Part IV — LLVM IR)** — Built on Ch 3's pipeline map and Ch 4's `Module`/`Function`/`BasicBlock`/`Instruction` API, `isa<>`/`dyn_cast<>`, and `InstVisitor`.
- **Ch 28–37 (Part V — Clang Frontend)** — Built on Ch 1's Clang subproject overview, Ch 3's driver/cc1 split and AST stage, and Ch 4's `LLVMContext` and diagnostic handler patterns.
- **Ch 38–43 (Part VI — Clang CodeGen)** — Built on Ch 3's LLVM IR emission stage and Ch 5's component model for linking against `libclang-cpp`.
- **Ch 44–51 (Part VII — Clang Multi-Language)** — Built on Ch 3's offload compilation section (CUDA, HIP, OpenMP target) and Ch 1's NVPTX and AMDGPU backend descriptions.
- **Ch 52–59 (Part VIII — ClangIR)** — Built on Ch 3's CIR stage description and Ch 1's ClangIR roadmap entry.
- **Ch 60–71 (Part X — Analysis and Middle End)** — Built on Ch 4's entire API chapter: ADT containers, `InstVisitor`, use-def traversal, `GraphTraits`, and `llvm::Error`; and on Ch 5's pass plugin infrastructure.
- **Ch 72–78 (Part XII — Polly)** — Built on Ch 3's opt pipeline and Ch 5's pass plugin loading mechanism.
- **Ch 79–86 (Part XIII — LTO and Whole-Program)** — Built on Ch 3's LTO and ThinLTO pipeline sections and Ch 2's `LLVM_ENABLE_LTO` build variable documentation.
- **Ch 87–95 (Part XIV — Backend)** — Built on Ch 3's MachineIR and SelectionDAG/GlobalISel pipeline stages and Ch 5's target registration pattern.
- **Ch 96–109 (Parts XV–XVI — Targets, JIT, Sanitizers)** — Built on Ch 1's compiler-rt and OrcJIT subproject descriptions, Ch 3's ORC JIT section, and Ch 5's component model for linking target libraries.
- **Ch 134–144 (Part XIX — MLIR Foundations)** — Built on Ch 3's MLIR pipeline, Ch 4's API idioms (reused verbatim in MLIR), and Ch 1's MLIR subproject overview.
- **Ch 196 (Part XXVI — Ecosystem Frontiers)** — Built on Ch 1's downstream ecosystem section and Ch 3's LLVM IR as interchange layer.
- **Ch 229–239 (Parts XXX–XXXI — AI-First PL Design, Frontier AI Evolution)** — Built on Ch 1's research roadmap and Ch 3's AI/ML-guided pipeline roadmap entries.

---

## Navigation

- (first part)
- → [Part II — Compiler Theory](../part-02-compiler-theory/)

---

*@copyright jreuben11*
