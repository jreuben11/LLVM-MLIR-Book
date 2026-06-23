# Chapter 231 — GraalVM: Native Image, Truffle Interpreters, and Polyglot Runtimes

*Part XXVIII — Language-Specific Compilation Ecosystems*

GraalVM is Oracle's polyglot virtual machine platform that unifies language implementation, JIT compilation, and ahead-of-time native compilation under a single framework. Its three pillars—the Graal JIT compiler written in Java, the Truffle language implementation framework, and SubstrateVM (Native Image)—enable a workflow where a language implementer writes a simple AST interpreter and receives an optimizing JIT, an ahead-of-time compiler, and interoperability with every other Truffle language at no extra cost. GraalVM's connection to LLVM is direct: Sulong executes LLVM bitcode on Truffle, making GraalVM a consumer of LLVM IR. This chapter covers the architecture of each layer, the mechanics of partial evaluation that makes Truffle fast, the closed-world assumption behind Native Image, and the Polyglot API that crosses language boundaries.

## Table of Contents

- [231.1 Architecture Overview](#2311-architecture-overview)
- [231.2 Truffle Framework: AST Interpretation and Specialization](#2312-truffle-framework-ast-interpretation-and-specialization)
  - [Node Specialization with @Specialization](#node-specialization-with-specialization)
  - [Assumptions and Deoptimization](#assumptions-and-deoptimization)
- [231.3 Partial Evaluation, Sea-of-Nodes IR, and JVMCI](#2313-partial-evaluation-sea-of-nodes-ir-and-jvmci)
  - [Partial Evaluation Mechanics](#partial-evaluation-mechanics)
  - [Sea-of-Nodes IR](#sea-of-nodes-ir)
  - [JVMCI: Java-Level Compiler Interface](#jvmci-java-level-compiler-interface)
- [231.4 SubstrateVM and Native Image](#2314-substratevm-and-native-image)
  - [Build-Time Pipeline](#build-time-pipeline)
  - [Points-to Analysis](#points-to-analysis)
  - [Heap Snapshotting](#heap-snapshotting)
  - [Limitations](#limitations)
- [231.5 Building a Language on Truffle](#2315-building-a-language-on-truffle)
  - [Frame Layout](#frame-layout)
- [231.6 GraalPy, TruffleRuby, GraalJS](#2316-graalpy-truffleruby-graaljs)
  - [Performance Characteristics](#performance-characteristics)
- [231.7 Polyglot API: Cross-Language Interoperability](#2317-polyglot-api-cross-language-interoperability)
  - [Interop Protocol](#interop-protocol)
- [231.8 Sulong: LLVM Bitcode on Truffle](#2318-sulong-llvm-bitcode-on-truffle)
  - [How Sulong Works](#how-sulong-works)
  - [Running LLVM Bitcode](#running-llvm-bitcode)
  - [Memory and Pointer Model](#memory-and-pointer-model)
- [Chapter Summary](#chapter-summary)

---

## 231.1 Architecture Overview

GraalVM Community Edition ([github.com/oracle/graal](https://github.com/oracle/graal)) stacks four layers:

```
┌─────────────────────────────────────────────────────────────┐
│  Guest Languages: GraalPy, TruffleRuby, GraalJS, GraalWasm  │
│  (Truffle AST interpreters, each ~20–50 KLOC)               │
├─────────────────────────────────────────────────────────────┤
│  Truffle Framework  (polyglot.truffle.api.*)                 │
│  @Specialization, PartialEvaluator, CallTarget, Assumptions  │
├─────────────────────────────────────────────────────────────┤
│  Graal JIT Compiler  (compiler/graal)                        │
│  Sea-of-Nodes IR, JVMCI interface, PE compilation           │
├─────────────────────────────────────────────────────────────┤
│  Substrate / JVM  (SubstrateVM for AOT, HotSpot for JIT)     │
└─────────────────────────────────────────────────────────────┘
```

When running on HotSpot (JVM mode), Truffle languages are JIT-compiled by the Graal compiler via JVMCI. When compiled with `native-image`, SubstrateVM replaces the JVM entirely, producing a standalone binary with millisecond startup times.

The canonical reference is Würthinger et al., "Practical Partial Evaluation for High-Performance Dynamic Language Runtimes," PLDI 2017.

## 231.2 Truffle Framework: AST Interpretation and Specialization

Truffle's central insight is that an AST interpreter, when subjected to *partial evaluation* with respect to a fixed program AST, specializes into code competitive with a hand-written optimizing compiler for that program. The language implementer writes nodes; Truffle handles the rest.

### Node Specialization with @Specialization

A Truffle `Node` subclass implements an `execute` method that dispatches based on runtime types. The `@Specialization` annotation generates a state machine that caches the type profile and avoids repeated type checks:

```java
@NodeChild("left")
@NodeChild("right")
public abstract class AddNode extends ExpressionNode {

    @Specialization
    protected long doLong(long left, long right) {
        return left + right;
    }

    @Specialization
    protected double doDouble(double left, double right) {
        return left + right;
    }

    @Specialization
    @TruffleBoundary  // prevents inlining into compiled code
    protected Object doGeneric(Object left, Object right) {
        // fallback: BigInteger, String concatenation, etc.
        return genericAdd(left, right);
    }
}
```

The `@NodeChild` annotations declare input nodes whose `execute` methods are called first. The Truffle DSL processor (`truffle/dsl_processor`) generates a concrete class `AddNodeGen` with a `state_0_` bitfield tracking which specializations have been activated.

State machine transitions:

```
Initial → [first long+long seen] → LONG state (monomorphic, fast)
LONG    → [double seen]          → LONG_DOUBLE (bimorphic)
LONG_DOUBLE → [other type]       → GENERIC (polymorphic)
```

Once in GENERIC, the node never de-specializes. If the monomorphic assumption is later invalidated (e.g., a different type arrives after compilation), the compiled code is *deoptimized* and interpretation resumes from the invalidated guard.

### Assumptions and Deoptimization

`Assumption` is a flag that starts true and can be invalidated:

```java
Assumption singleContextAssumption =
    Truffle.getRuntime().createAssumption("single context");

// In compiled code, this becomes a guard:
if (!singleContextAssumption.isValid()) deoptimize();
```

When `singleContextAssumption.invalidate()` is called (e.g., a second language context is created), all compiled code that relied on it is deoptimized—control returns to the interpreter at the exact bytecode/AST node where execution was, and the interpreter re-specializes with the invalidated assumption absent.

Source: [`truffle/api/src/com/oracle/truffle/api/`](https://github.com/oracle/graal/tree/master/truffle/src/com.oracle.truffle.api/src/com/oracle/truffle/api)

## 231.3 Partial Evaluation, Sea-of-Nodes IR, and JVMCI

The Graal JIT compiler is invoked on a `CallTarget` (a compiled Truffle function). Partial evaluation (PE) inlines the interpreter dispatch loop against the fixed AST tree, reducing the interpreter overhead to nothing: the compiled code is specialized for the specific program being run.

### Partial Evaluation Mechanics

PE works by treating the AST nodes as *compile-time constants*. The Truffle PE engine:

1. Starts from `CallTarget.call()`
2. Inlines `execute()` on the root node
3. Recursively inlines child node `execute()` calls, with the node identity as a compile-time constant
4. Folds type checks that have been resolved by `@Specialization` state
5. Stops inlining at `@TruffleBoundary` annotations

The result is a straight-line (or lightly branched) IR graph that looks like hand-specialized native code for the given program shape.

### Sea-of-Nodes IR

The Graal compiler uses a *sea-of-nodes* representation where both data and control dependencies are explicit graph edges. Unlike LLVM's CFG + def-use chains, in sea-of-nodes:

- Control nodes (Begin, End, If, Loop) form a control subgraph
- Data nodes (Add, Load, Phi) float freely, pinned only where they must appear relative to effects
- Scheduling is deferred to late in compilation, after optimizations reduce node count

This representation simplifies global value numbering (GVN), loop-invariant code motion, and escape analysis because floating nodes are trivially movable.

Key IR classes (in `compiler/src/org.graalvm.compiler.nodes/`):

| Class | Role |
|-------|------|
| `ValueNode` | Base for all data-producing nodes |
| `FixedNode` | Nodes with a fixed control position (side effects) |
| `IfNode` | Two-way control split |
| `PhiNode` | Value merge at control joins |
| `LoopBeginNode` / `LoopEndNode` | Loop structure |
| `InvokeNode` | Method calls (virtual + devirtualized) |

### JVMCI: Java-Level Compiler Interface

JVMCI (JVM Compiler Interface, [JEP 243](https://openjdk.org/jeps/243)) is the HotSpot API that allows an external JIT compiler (Graal) to replace HotSpot's C2 compiler for a specific method:

```java
// Graal registers itself via ServiceLoader
@ServiceProvider(JVMCICompiler.class)
public class GraalJVMCICompiler implements JVMCICompiler {
    @Override
    public CompilationRequestResult compileMethod(CompilationRequest request) {
        // translate HotSpot's StructuredGraph to Graal's IR,
        // optimize, emit machine code via HotSpot's nmethod infrastructure
    }
}
```

When HotSpot decides to compile a method (invocation count threshold), it calls JVMCI instead of C2. Graal then emits an `nmethod` and registers it. This means Graal is itself JIT-compiled by HotSpot—a *metacircular* arrangement where Graal compiles Graal.

## 231.4 SubstrateVM and Native Image

Native Image (`native-image` command) compiles a JVM application to a standalone native binary using SubstrateVM as the replacement runtime. It performs whole-program analysis at build time under a *closed-world assumption*: all reachable code must be analyzable statically.

### Build-Time Pipeline

```
JVM bytecode (all jars)
    │
    ▼
Points-to Analysis (Bigbang)
    │  determines: reachable types, methods, fields
    ▼
Universe (closed set of reachable program elements)
    │
    ▼
Graal AOT Compilation (each reachable method → machine code)
    │
    ▼
Heap Snapshotting (pre-initialize objects into the image heap)
    │
    ▼
native binary (.elf / .macho)
```

### Points-to Analysis

SubstrateVM's points-to analysis (`Bigbang`) is a context-insensitive, flow-insensitive analysis that determines for every heap variable which types it can point to. It is modeled as a constraint-propagation problem over type-flow graphs:

```
type_set(v) ⊇ type_set(source)  for every assignment v = source
type_set(v) ⊇ alloc_type         for every `new T()` assigned to v
virtual_dispatch(call, type) → reachable methods
```

Reflection, `Class.forName`, `Method.invoke`, and dynamic proxies break the closed-world assumption. They require build-time configuration files (JSON) that enumerate all dynamically referenced classes:

```json
// reflect-config.json
[
  {
    "name": "com.example.MyService",
    "allDeclaredConstructors": true,
    "allPublicMethods": true
  }
]
```

The `native-image-agent` can generate these files by instrumenting a JVM run.

### Heap Snapshotting

Static initializers (`<clinit>`) that run at class initialization time are executed *at build time*. The resulting heap objects are serialized into the image heap section of the binary and mapped read-only at startup. This eliminates JVM class loading and initialization at runtime, contributing to sub-10ms startup.

```bash
native-image \
  --no-fallback \
  -H:+ReportExceptionStackTraces \
  -H:ReflectionConfigurationFiles=reflect-config.json \
  --initialize-at-build-time=org.slf4j \
  -jar myapp.jar \
  -o myapp-native
```

### Limitations

| Feature | JVM | Native Image |
|---------|-----|--------------|
| Reflection | Full | Config required |
| Dynamic class loading | Yes | No (closed world) |
| JNI | Yes | Config required |
| Finalizers | Supported | Deprecated |
| JVMTI / agents | Full | Limited |
| Startup time | 50–500ms | 5–50ms |
| Memory footprint | High (JIT structures) | Low (no JIT) |
| Peak throughput | High (profile-guided JIT) | Moderate (AOT only) |

Project Leyden ([JEP 483](https://openjdk.org/jeps/483)) pursues a partially-closed-world AOT approach for OpenJDK that relaxes the closed-world requirement.

## 231.5 Building a Language on Truffle

A minimal Truffle language requires:

1. A `TruffleLanguage` subclass annotated with `@TruffleLanguage.Registration`
2. A parser producing a tree of `Node` subclasses
3. An `ExecuteContext` holding per-language runtime state
4. An `execute` method on each node

```java
@TruffleLanguage.Registration(
    id = "simplelang",
    name = "SimpleLang",
    version = "0.1",
    defaultMimeType = "application/x-simplelang",
    characterMimeTypes = "application/x-simplelang"
)
public class SimpleLang extends TruffleLanguage<SimpleLangContext> {

    @Override
    protected SimpleLangContext createContext(Env env) {
        return new SimpleLangContext(env);
    }

    @Override
    protected CallTarget parse(ParsingRequest request) throws Exception {
        Source source = request.getSource();
        Node rootNode = new Parser(this).parse(source.getCharacters().toString());
        return rootNode.getCallTarget();
    }
}
```

Nodes access their language context via `lookupContextReference(SimpleLang.class).get(this)`. The context reference is optimized by PE to a constant if only one language instance exists (`singleContextAssumption`).

### Frame Layout

Truffle `VirtualFrame` provides typed slot access for local variables:

```java
// Declare slots in frame descriptor
FrameDescriptor descriptor = FrameDescriptor.newBuilder()
    .addSlot(FrameSlotKind.Long, "x", null)
    .addSlot(FrameSlotKind.Long, "y", null)
    .build();

// In a node's execute:
long x = frame.getLong(xSlot);
frame.setLong(ySlot, x + 1);
```

Frame slots with stable type profiles are optimized by PE to typed register accesses; unstable slots fall back to `Object` boxing.

## 231.6 GraalPy, TruffleRuby, GraalJS

Each Truffle guest language is an interpreter in a separate repository that delegates JIT and AOT to the Truffle framework:

| Language | Repository | Python/Ruby/JS Standard Library |
|----------|-----------|----------------------------------|
| GraalPy | [oracle/graalpython](https://github.com/oracle/graalpython) | CPython C extension compatibility via HPy / Cython |
| TruffleRuby | [oracle/truffleruby](https://github.com/oracle/truffleruby) | Ruby stdlib + C extensions via Sulong |
| GraalJS | [oracle/graaljs](https://github.com/oracle/graaljs) | Node.js compatibility; V8 API subset |
| GraalWasm | [oracle/graal/wasm](https://github.com/oracle/graal/tree/master/wasm) | Wasm MVP + GC + tail-call proposals |

### Performance Characteristics

TruffleRuby achieves near-MRI-Ruby throughput for pure Ruby code and surpasses it for long-running workloads once the JIT warms up. The warm-up curve is the principal trade-off:

- Interpretation: ~5–20× slower than JIT-compiled
- After 1000–5000 invocations: JIT kicks in, approaches near-native speeds
- With Native Image: fast startup but no runtime JIT (AOT quality only)

GraalJS performance on the JVM is comparable to V8 for compute-intensive benchmarks; GraalPy targets CPython compatibility over peak performance.

## 231.7 Polyglot API: Cross-Language Interoperability

GraalVM's Polyglot API ([org.graalvm.polyglot](https://www.graalvm.org/sdk/javadoc/)) allows host code (Java/Kotlin) to execute guest language code and exchange values without serialization:

```java
try (Context ctx = Context.newBuilder()
        .allowAllAccess(true)
        .build()) {

    // Execute Python
    ctx.eval("python", "import math; result = math.sqrt(2.0)");
    double result = ctx.getBindings("python").getMember("result").asDouble();

    // Execute JavaScript
    Value jsFunc = ctx.eval("js", "(x) => x * x");
    int squared = jsFunc.execute(7).asInt();

    // Pass Java object to Python
    ctx.getBindings("python").putMember("javaList",
        ctx.asValue(List.of(1, 2, 3)));
}
```

### Interop Protocol

Truffle's `InteropLibrary` defines the cross-language object protocol. Any object can advertise capabilities by implementing library messages:

```java
@ExportLibrary(InteropLibrary.class)
public class SimpleLangArray implements TruffleObject {

    @ExportMessage
    public boolean hasArrayElements() { return true; }

    @ExportMessage
    public long getArraySize() { return data.length; }

    @ExportMessage
    public Object readArrayElement(long index)
            throws InvalidArrayIndexException {
        if (index < 0 || index >= data.length)
            throw InvalidArrayIndexException.create(index);
        return data[(int) index];
    }
}
```

When Python code iterates a Java `List` passed through `putMember`, Python's `for` loop calls `InteropLibrary.getArraySize()` and `readArrayElement()` on the Java wrapper—zero serialization, zero copies.

## 231.8 Sulong: LLVM Bitcode on Truffle

Sulong runs LLVM IR / bitcode on the Truffle AST interpreter, enabling C/C++ code to execute inside GraalVM and interoperate with other Truffle languages. This is GraalVM's primary integration point with the LLVM ecosystem.

### How Sulong Works

```
C/C++ source
    │
    ▼ clang --emit-llvm
LLVM bitcode (.bc)
    │
    ▼ lli (GraalVM lli, not LLVM lli)
Sulong LLVM IR Interpreter (Truffle nodes per LLVM instruction)
    │
    ▼ Truffle PE + Graal JIT
JIT-compiled native code
```

Each LLVM instruction has a corresponding Truffle node: `LLVMLongAddNode`, `LLVMLongMulNode`, `LLVMGetElementPtrNode`, etc. These are in [`sulong/projects/com.oracle.truffle.llvm.runtime/`](https://github.com/oracle/graal/tree/master/sulong/projects/com.oracle.truffle.llvm.runtime).

### Running LLVM Bitcode

```bash
# Compile to bitcode
clang -emit-llvm -c -O1 mylib.c -o mylib.bc

# Run with GraalVM lli (must be GraalVM, not standard LLVM lli)
lli --llvm.managed mylib.bc

# Or from Java using Polyglot API
ctx.eval(Source.newBuilder("llvm", new File("mylib.bc")).build());
```

### Memory and Pointer Model

Sulong implements two memory models:

- **Native memory** (`--llvm.managed=false`): pointers are actual OS memory addresses; allows unsafe pointer arithmetic, FFI
- **Managed memory** (`--llvm.managed=true`): pointers are Java object references; enables GC integration and safe interop but restricts raw pointer arithmetic

Managed mode allows Sulong to interoperate with GraalPy: a Python `bytearray` can be passed to a C function as a managed pointer, with Truffle tracking the object's liveness for GC.

TruffleRuby uses Sulong for C extension execution: `require 'openssl'` loads the OpenSSL C extension as LLVM bitcode, interpreted by Sulong, and optimized alongside Ruby code by the same Graal JIT.

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **GraalVM 24.x / JDK 25 integration**: GraalVM Community tracks OpenJDK releases closely; GraalVM 24.2 (paired with JDK 25) is expected to ship JVMCI improvements that reduce PE compilation latency and expose new intrinsic hooks for Truffle specialization caching — follow [oracle/graal milestones](https://github.com/oracle/graal/milestones).
- **Project Leyden Phase 2 (JEP 483)**: OpenJDK's Leyden AOT layer is iterating toward a "condensed JDK" that stores a pre-linked class hierarchy; GraalVM Native Image's Bigbang points-to analysis will need to consume or produce Leyden-compatible condensed images, and co-ordination discussions are ongoing on the [leyden-dev mailing list](https://mail.openjdk.org/pipermail/leyden-dev/).
- **Sulong managed-pointer arithmetic relaxations**: The Sulong team is extending managed-memory mode to support more LLVM pointer idioms (integer-to-pointer casts, union-like struct reinterpretation) required by real C extension codebases; progress tracked in [graal/sulong issue #1062](https://github.com/oracle/graal/issues?q=label%3Asulong).
- **GraalPy HPy ABI stabilization**: GraalPy's C extension compatibility layer targets the HPy 0.9 stable ABI; patches landing in [hpyproject/hpy](https://github.com/hpyproject/hpy) reduce the CPython-only surface that forces fallback to slow CPython emulation mode.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Truffle speculation-free partial evaluation**: Research prototypes from Oracle Labs and TU Wien explore eliminating deoptimization stacks entirely via ahead-of-time recompilation triggers; this would allow Native Image binaries to re-specialize without JIT infrastructure, narrowing the throughput gap vs. JVM mode described in §231.6.
- **LLVM bitcode as primary Sulong IR (LLVM 18+ opaque-pointer convergence)**: Sulong currently lowers opaque LLVM pointers to managed objects with special rules; as LLVM's opaque-pointer model stabilizes across LLVM 17–22, Sulong's IR ingestion layer is being rewritten to use a single typed-value abstraction, eliminating the dual native/managed memory configuration flag (`--llvm.managed`) in favor of per-allocation policy.
- **GraalWasm ahead-of-time tier with Wasm GC proposal**: GraalWasm targets the W3C Wasm GC proposal (finalized 2023, still ramping in runtimes); full AOT compilation of Wasm GC programs via SubstrateVM requires teaching Bigbang to reason about Wasm's heap type hierarchy — active work under [graal/wasm](https://github.com/oracle/graal/tree/master/wasm).
- **Polyglot isolates and shared-heap multi-tenancy**: The current Polyglot `Context` model creates per-context heaps; Oracle Labs research into *shared-heap polyglot* (where GraalPy and TruffleRuby objects live in the same GC heap) is targeting production readiness for SaaS multi-tenant deployments — related to the "Value Sharing" RFC posted to the Truffle issue tracker in late 2024.

### 5-Year Horizon (Long-Term, by ~2031)

- **Full metacompilation stack in a single binary**: The vision of embedding Graal's PE engine inside a Native Image binary (so a native app can JIT its own Truffle plugins at runtime without a JVM) has been prototyped in the "SVM JIT" research line; productizing this would collapse the JVM-mode vs. Native Image trade-off and is a multi-year research program at Oracle Labs.
- **MLIR-based Graal IR back-end**: Community proposals have explored replacing the sea-of-nodes Graal IR with an MLIR-based representation that would allow Graal to share LLVM middle-end passes (e.g., polyhedral loop optimizations via Polly) and emit to LLVM's machine-code back-ends directly; this would make Sulong redundant for C code and enable GraalVM languages to target GPU/accelerator back-ends via MLIR's `gpu` and `nvgpu` dialects.
- **Verifiable partial evaluation via Lean/Coq proofs**: Following CompCert and Vellvm precedents, long-term academic work aims to formally verify Truffle's PE correctness — that the compiled output of partial evaluation is observationally equivalent to the interpreted AST execution — using Lean 4 as the proof assistant; early formalization work is appearing in POPL/PLDI venues.

---

## Chapter Summary

- GraalVM stacks Truffle (AST interpreter framework), Graal JIT (sea-of-nodes, JVMCI), and SubstrateVM (AOT native image) on a common foundation
- **Truffle `@Specialization`** generates a type-specializing state machine; activated specializations are compiled by partial evaluation against the fixed program AST
- **Partial evaluation** treats AST node identity as compile-time constant and inlines interpreter dispatch into program-specific machine code; `@TruffleBoundary` stops inlining at interpreter/runtime boundaries
- **JVMCI** allows Graal (written in Java) to replace HotSpot's C2 JIT via a stable API; Graal is itself JIT-compiled metacircularly
- **Native Image** performs whole-program points-to analysis (Bigbang), executes static initializers at build time (heap snapshot), and emits a standalone binary with sub-10ms startup at the cost of a closed-world assumption
- Reflection, dynamic class loading, and JNI require explicit build-time configuration files; `native-image-agent` generates these from a profiling run
- **GraalPy/TruffleRuby/GraalJS** are production Truffle interpreters; interop via `InteropLibrary` enables zero-copy cross-language object passing
- **Sulong** executes LLVM bitcode on Truffle, making GraalVM an LLVM IR consumer; TruffleRuby uses Sulong for C extension support

**Cross-references**: [Chapter 193 — Julia, LLVM, and Type Specialization](ch193-julia-llvm-specialization.md) | [Chapter 230 — Cranelift: A Lightweight JIT for WebAssembly and Rust](ch230-cranelift-lightweight-jit.md) | [Chapter 106 — WebAssembly and BPF](../part-15-targets/ch106-webassembly-and-bpf.md)

**References**:
- Würthinger et al., "Practical Partial Evaluation for High-Performance Dynamic Language Runtimes," PLDI 2017. [ACM](https://dl.acm.org/doi/10.1145/3062341.3062381)
- [GraalVM Documentation](https://www.graalvm.org/latest/docs/)
- [Native Image Build Overview](https://www.graalvm.org/latest/reference-manual/native-image/basics/)
- [Truffle Language Implementation Tutorial](https://www.graalvm.org/latest/graalvm-as-a-platform/language-implementation-framework/LanguageTutorial/)
- [Sulong: LLVM Bitcode on GraalVM](https://github.com/oracle/graal/tree/master/sulong)
- [JVMCI JEP 243](https://openjdk.org/jeps/243)
