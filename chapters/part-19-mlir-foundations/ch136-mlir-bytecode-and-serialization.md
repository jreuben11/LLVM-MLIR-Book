# Chapter 136 — MLIR Bytecode and Serialization

*Part XIX — MLIR Foundations*

MLIR IR lives in two serialized forms: a textual format designed for human readability and debuggability, and a binary bytecode format designed for speed and compactness. Understanding both forms — their structure, their APIs, and the tools that convert between them — is essential for anyone building production MLIR-based compilers where startup time, caching, and interoperability matter. This chapter covers the textual format's parsing and printing model, the bytecode format's section-based binary layout and version history, the C++ APIs for reading and writing both forms, per-dialect custom encoding through `BytecodeDialectInterface`, external resource blobs for large data, the `mlir-translate` tool for cross-IR conversion, and practical patterns for production pipelines that use `.mlirbc` files as compiler artifacts.

---

## 136.1 The Textual Format

### Design Principles

MLIR's textual format is its primary debugging and testing surface. Every `.mlir` file that appears in test suites, LLVM's `lit` infrastructure, or online documentation is in this format. The design goals are:

- **Exact round-trip**: printing a module and re-parsing it produces an identical IR (modulo SSA value name choices)
- **Self-contained**: a `.mlir` file embeds all type, attribute, and operation information without external schema references
- **Extensible by dialects**: each dialect registers its own printer/parser via `OpAsmInterface` and the asm dialect interface hooks

### Syntax Elements

The textual format uses a small set of punctuation conventions that appear throughout every MLIR file:

| Element | Syntax | Example |
|---------|--------|---------|
| SSA values | `%name` or `%N` | `%arg0`, `%3` |
| Block labels | `^bbN:` | `^bb0:`, `^then:` |
| Region delimiters | `{` ... `}` | enclosing a `func.func` body |
| Op result list | `%a, %b = op.name ...` | `%lo, %hi = arith.divui` |
| Attribute dict | `{key = value, ...}` | `{sym_name = "main"}` |
| Type annotation | `: type` after values | `%0 : i32`, `-> (i32, f32)` |
| Location info | `loc(...)` suffix | `loc("file.mlir":10:5)` |

### Builtin and Dialect Types

Type syntax is dialect-extensible. Builtin types use unqualified names:

```mlir
// Integer types: signless, signed, unsigned
%a : i32
%b : si64    // signed 64-bit
%c : ui8     // unsigned 8-bit
%d : index   // target-width index type

// Float types
%e : f32
%f : bf16
%g : f8E4M3FN  // narrowed float for ML

// Shaped types
%h : tensor<4x8xf32>
%i : tensor<?x?xf32>               // dynamic dimensions
%j : memref<4x8xf32>
%k : memref<?xf32, strided<[1], offset: ?>>  // strided layout

// Dialect types use the !dialect.typename syntax:
%l : !llvm.ptr
%m : !llvm.struct<(i32, f64)>
%n : !transform.any_op
```

### Attribute Syntax

Attributes carry compile-time constants and metadata:

```mlir
// Numeric literals (type-suffixed)
{value = 42 : i32}
{value = 3.14 : f32}

// Dense element constants
{matrix = dense<[[1, 2], [3, 4]]> : tensor<2x2xi32>}
{weights = dense<0.0> : tensor<4x8xf32>}   // splat

// Dense resource reference (blob data stored separately)
{weights = dense_resource<my_weights> : tensor<1024x512xf16>}

// String attributes
{sym_name = "main", comment = "entry point"}

// Array attributes
{dims = [1, 8, 16] : array<i64>}
```

### Parsing APIs

The C++ parser entry points are in [`mlir/include/mlir/Parser/Parser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Parser/Parser.h). All overloads auto-detect whether the input is textual or bytecode:

```cpp
#include "mlir/Parser/Parser.h"
#include "mlir/Dialect/Arith/IR/Arith.h"
#include "mlir/Dialect/Func/IR/FuncOps.h"

mlir::MLIRContext ctx;
ctx.loadDialect<mlir::arith::ArithDialect, mlir::func::FuncDialect>();

mlir::ParserConfig config(&ctx);

// Parse from a file path (auto-detects text vs bytecode)
mlir::OwningOpRef<mlir::ModuleOp> mod =
    mlir::parseSourceFile<mlir::ModuleOp>("/path/to/input.mlir", config);

// Parse from a string (for tests and generated IR)
llvm::StringRef src = R"mlir(
  func.func @add(%a: i32, %b: i32) -> i32 {
    %c = arith.addi %a, %b : i32
    return %c : i32
  }
)mlir";
mlir::OwningOpRef<mlir::ModuleOp> mod2 =
    mlir::parseSourceString<mlir::ModuleOp>(src, config);

// Parse with a SourceMgr (for diagnostics with file:line:col info)
auto sourceMgr = std::make_shared<llvm::SourceMgr>();
sourceMgr->AddNewSourceBuffer(
    llvm::MemoryBuffer::getFile("/path/to/input.mlir").get(), llvm::SMLoc{});
mlir::OwningOpRef<mlir::ModuleOp> mod3 =
    mlir::parseSourceFile<mlir::ModuleOp>(sourceMgr, config);
```

### Printing APIs

The primary printing paths go through [`mlir/include/mlir/IR/OpImplementation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/OpImplementation.h) and the `Operation::print` family:

```cpp
// Print entire module to stderr (for debugging):
module->dump();

// Print to a stream with options:
mlir::OpPrintingFlags flags;
flags.enableDebugInfo(/*prettyForm=*/true);
flags.printGenericOpForm(false);
flags.elideLargeElementsAttrs(/*largeAttrLimit=*/64);

module->print(llvm::outs(), flags);

// Print a single operation without its surrounding context:
op->print(llvm::errs(), mlir::OpPrintingFlags().useLocalScope());
```

The `--mlir-print-local-scope` mlir-opt flag corresponds to `useLocalScope()` — it avoids printing large dense attributes in full, substituting a reference instead, which is essential for human-readable dumps of ML models.

---

## 136.2 The Bytecode Format Overview

### Design Goals

The MLIR bytecode format (`.mlirbc`) was designed to solve three problems with the textual format in production use:

1. **Parse time**: textual parsing involves tokenizing, string-based op lookup, and attribute parsing; bytecode uses integer IDs everywhere and skips tokenization entirely
2. **File size**: textual IR repeats op names and type strings thousands of times; bytecode uses string table references
3. **Lazy loading**: the bytecode reader supports skipping isolated regions until they are needed, enabling tools to work on subsets of large modules

In practice, bytecode files are typically 3–5× smaller than their textual equivalents and parse 8–12× faster on large ML models (millions of ops). The trade-off is opacity: bytecode is not human-readable without tooling.

### File Header

All MLIR bytecode files begin with a 4-byte magic signature:

```
4d 4c ef 52  →  "ML\xefR"
```

The `\xef` byte prevents the file from being accidentally treated as ASCII or UTF-8 text (it would form an invalid UTF-8 sequence in that position). Immediately following the magic is a varint-encoded format version number and a null-terminated producer string (e.g., `"MLIR22.1.3"`). The `isBytecode()` function in [`mlir/include/mlir/Bytecode/BytecodeReader.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Bytecode/BytecodeReader.h) checks only these 4 magic bytes.

### Section Layout

After the header, the file is organized into named sections whose IDs are defined in [`mlir/include/mlir/Bytecode/Encoding.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Bytecode/Encoding.h):

| Section ID | Name | Contents |
|------------|------|----------|
| 0 | `kString` | All string data: op names, dialect names, attribute keys, symbol names; null-terminated, in one contiguous buffer |
| 1 | `kDialect` | Dialect name references (as string table indices) plus per-dialect custom binary data |
| 2 | `kAttrType` | Encoded attribute and type objects; each entry uses a dialect's `BytecodeDialectInterface` or falls back to textual-in-binary |
| 3 | `kAttrTypeOffset` | An offset table into section 2, enabling random-access to attribute/type entries |
| 4 | `kIR` | The operation tree: each op is a bitmask byte followed by result types, operand indices, attribute indices, successor block IDs, and region data |
| 5 | `kResource` | Binary blob data for external resources (e.g., weight tensors) |
| 6 | `kResourceOffset` | Offset table into section 5 for named blobs |
| 7 | `kDialectVersions` | Per-dialect version information for forward/backward compatibility |
| 8 | `kProperties` | Native properties encoding (LLVM 22 only) |

Each section is preceded by its length as a varint, so readers can skip unknown sections.

### Encoding Primitives

All multi-byte integers use LLVM's varint encoding: values 0–127 encode in one byte; larger values use the high bit of each byte as a continuation flag. This is the same encoding used throughout LLVM's bitcode format. The IR section uses an operation encoding mask (`OpEncodingMask`) as the first byte of each op, with bits indicating which components are present:

```
kHasAttrs         = 0b00000001
kHasResults       = 0b00000010
kHasOperands      = 0b00000100
kHasSuccessors    = 0b00001000
kHasInlineRegions = 0b00010000
kHasUseListOrders = 0b00100000
kHasProperties    = 0b01000000
```

An op with no attributes, no successors, and no regions encodes to a mask byte of `0b00000110` (has results + has operands) followed by result type indices and operand value indices — typically 3–6 bytes total for a simple arithmetic op.

---

## 136.3 Writing and Reading Bytecode

### Writing Bytecode

The write path is in [`mlir/include/mlir/Bytecode/BytecodeWriter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Bytecode/BytecodeWriter.h). The primary entry point is `writeBytecodeToFile`:

```cpp
#include "mlir/Bytecode/BytecodeWriter.h"

// Write with default settings (current version, MLIR producer string):
std::string buffer;
llvm::raw_string_ostream os(buffer);
if (failed(mlir::writeBytecodeToFile(module.get(), os)))
  return failure();

// Write to a file stream:
std::error_code ec;
llvm::raw_fd_ostream fileStream("output.mlirbc", ec,
                                llvm::sys::fs::OF_None);
if (ec) { /* handle error */ }

mlir::BytecodeWriterConfig config("MyCompiler-1.0");
config.setDesiredBytecodeVersion(6);  // kVersion = 6 in LLVM 22

if (failed(mlir::writeBytecodeToFile(module.get(), fileStream, config)))
  return failure();
```

The `BytecodeWriterConfig` constructor takes an optional producer string (stored in the file header, has no effect on semantics). `setDesiredBytecodeVersion` requests a specific version; the writer returns failure if it cannot honor the request (e.g., requesting version 0 when the IR uses version-6 features like native properties).

### Reading Bytecode

Reading uses the same `parseSourceFile` / `parseSourceString` APIs as the textual format — auto-detection checks the magic bytes. For more control, the `BytecodeReader` class enables lazy loading:

```cpp
#include "mlir/Bytecode/BytecodeReader.h"
#include "mlir/Parser/Parser.h"

// Auto-detect and parse (simplest path):
mlir::OwningOpRef<mlir::ModuleOp> module =
    mlir::parseSourceFile<mlir::ModuleOp>("model.mlirbc", config);

// Manual bytecode read with lazy loading:
auto buffer = llvm::MemoryBuffer::getFile("model.mlirbc");
mlir::BytecodeReader reader(buffer->get()->getMemBufferRef(), config,
                            /*lazyLoad=*/true);

mlir::Block block;
if (failed(reader.readTopLevel(
    &block,
    // Callback: decide which ops to materialize immediately
    [](mlir::Operation *op) -> bool {
      // Only eagerly load func.func ops named "main"
      if (auto fn = llvm::dyn_cast<mlir::func::FuncOp>(op))
        return fn.getSymName() == "main";
      return false;
    })))
  return failure();

// Later, materialize a specific op on demand:
if (reader.isMaterializable(targetOp))
  if (failed(reader.materialize(targetOp)))
    return failure();

// Finalize: delete or materialize all remaining lazy ops
if (failed(reader.finalize([](mlir::Operation *) { return false; })))
  return failure();
```

Lazy loading is particularly valuable when working with large ML model modules where only a few functions need to be compiled in a given incremental build step.

### Command-Line Workflow

```bash
# Write bytecode:
/usr/lib/llvm-22/bin/mlir-opt --emit-bytecode input.mlir -o output.mlirbc

# Write bytecode at a specific version:
/usr/lib/llvm-22/bin/mlir-opt --emit-bytecode --emit-bytecode-version=5 \
  input.mlir -o output_v5.mlirbc

# Read bytecode (auto-detected) and apply a pass:
/usr/lib/llvm-22/bin/mlir-opt output.mlirbc --canonicalize -o result.mlir

# Inspect the magic bytes:
xxd output.mlirbc | head -1
# 4d4c ef52  → "ML\xefR" (confirmed MLIR bytecode)

# Round-trip test:
/usr/lib/llvm-22/bin/mlir-opt input.mlir --emit-bytecode | \
  /usr/lib/llvm-22/bin/mlir-opt --canonicalize > /dev/null
```

---

## 136.4 Bytecode Versioning

### Version History

The bytecode version is defined in `bytecode::BytecodeVersion` in [`mlir/include/mlir/Bytecode/Encoding.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Bytecode/Encoding.h). The current version in LLVM 22.1.x is **6** (`kVersion = 6`). The version milestones:

| Version | LLVM Release | Feature Added |
|---------|-------------|---------------|
| 0 (`kMinSupportedVersion`) | LLVM 15 | Initial binary format; string table, basic IR section |
| 1 (`kDialectVersioning`) | LLVM 16 | Per-dialect version information in `kDialectVersions` section |
| 2 (`kLazyLoading`) | LLVM 17 | Lazy loading of isolated regions |
| 3 (`kUseListOrdering`) | LLVM 18 | Encoding of use-list ordering for reproducibility |
| 4 (`kElideUnknownBlockArgLocation`) | LLVM 19 | Omit unknown locations on block args (compression) |
| 5 (`kNativePropertiesEncoding`) | LLVM 20 | Properties section; separate from discardable attrs |
| 6 (`kNativePropertiesODSSegmentSize`) | LLVM 21/22 | ODS segment size as native property |

### Compatibility Policy

MLIR's bytecode compatibility policy:

- **Forward compatibility** (new reader, old file): the reader supports all versions back to `kMinSupportedVersion` (0). A LLVM 22 tool can read LLVM 15 bytecode.
- **Backward compatibility** (old reader, new file): an older reader may fail on newer version files if those files use features from a higher version. The version byte is checked first and an error is emitted before attempting to read sections that don't exist.
- **Dialect version compatibility**: individual dialects may evolve their type/attribute encoding independently via `BytecodeDialectInterface::readVersion` / `writeVersion` (see Section 136.5).

The `BytecodeWriterConfig::setDesiredBytecodeVersion` method requests writing at an older version to maximize compatibility:

```cpp
// Write at version 3 for consumption by an LLVM 18 tool:
mlir::BytecodeWriterConfig compat;
compat.setDesiredBytecodeVersion(3);
if (failed(mlir::writeBytecodeToFile(module.get(), os, compat))) {
  // Failure means the IR uses features (e.g., native properties) that
  // cannot be represented at version 3.
  llvm::errs() << "Cannot write at requested version\n";
}
```

---

## 136.5 DialectBytecodeInterface

### Purpose and Registration

Dialects that define custom types or attributes can provide optimized binary encodings through `BytecodeDialectInterface`, declared in [`mlir/include/mlir/Bytecode/BytecodeImplementation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Bytecode/BytecodeImplementation.h). Without this interface, any dialect-defined type or attribute falls back to embedding its full textual form as a string inside the binary file — which is correct but wastes space.

Registration in the dialect's `initialize()` method:

```cpp
#include "mlir/Bytecode/BytecodeImplementation.h"

class MyDialectBytecodeInterface
    : public mlir::BytecodeDialectInterface {
public:
  using BytecodeDialectInterface::BytecodeDialectInterface;

  mlir::Attribute readAttribute(
      mlir::DialectBytecodeReader &reader) const override;
  mlir::Type readType(
      mlir::DialectBytecodeReader &reader) const override;
  mlir::LogicalResult writeAttribute(
      mlir::Attribute attr,
      mlir::DialectBytecodeWriter &writer) const override;
  mlir::LogicalResult writeType(
      mlir::Type type,
      mlir::DialectBytecodeWriter &writer) const override;
};

void MyDialect::initialize() {
  // ... add ops, types, attrs ...
  addInterface<MyDialectBytecodeInterface>(*this);
}
```

### Writing Types and Attributes

The writer interface uses discriminator integers to identify which type/attribute variant is being encoded:

```cpp
mlir::LogicalResult MyDialectBytecodeInterface::writeType(
    mlir::Type type,
    mlir::DialectBytecodeWriter &writer) const {
  return llvm::TypeSwitch<mlir::Type, mlir::LogicalResult>(type)
    .Case<MyVectorType>([&](MyVectorType vt) {
      // Discriminator byte: 0 = MyVectorType
      writer.writeVarInt(0);
      writer.writeVarInt(vt.getNumElements());
      writer.writeType(vt.getElementType());
      return mlir::success();
    })
    .Case<MyStructType>([&](MyStructType st) {
      writer.writeVarInt(1);
      writer.writeVarInt(st.getNumFields());
      for (auto fieldTy : st.getFieldTypes())
        writer.writeType(fieldTy);
      return mlir::success();
    })
    .Default([](mlir::Type) { return mlir::failure(); });
}
```

When `writeType` returns `failure()`, the bytecode writer automatically falls back to embedding the type's textual form. This makes it safe to add new type variants incrementally: unhandled variants degrade to textual-in-binary rather than causing an error.

### Reading Types and Attributes

```cpp
mlir::Type MyDialectBytecodeInterface::readType(
    mlir::DialectBytecodeReader &reader) const {
  uint64_t tag;
  if (failed(reader.readVarInt(tag)))
    return mlir::Type();

  switch (tag) {
  case 0: {  // MyVectorType
    uint64_t numElems;
    mlir::Type elemTy;
    if (failed(reader.readVarInt(numElems)) ||
        failed(reader.readType(elemTy)))
      return mlir::Type();
    return MyVectorType::get(reader.getContext(), numElems, elemTy);
  }
  case 1: {  // MyStructType
    uint64_t numFields;
    if (failed(reader.readVarInt(numFields)))
      return mlir::Type();
    llvm::SmallVector<mlir::Type> fieldTypes(numFields);
    for (auto &ty : fieldTypes)
      if (failed(reader.readType(ty)))
        return mlir::Type();
    return MyStructType::get(reader.getContext(), fieldTypes);
  }
  default:
    reader.emitError() << "unknown type tag: " << tag;
    return mlir::Type();
  }
}
```

### Dialect Versioning

Dialects can version their bytecode encoding independently of the global bytecode version, enabling smooth evolution:

```cpp
struct MyDialectVersion : public mlir::DialectVersion {
  uint64_t major = 0, minor = 0;
};

void MyDialectBytecodeInterface::writeVersion(
    mlir::DialectBytecodeWriter &writer) const {
  writer.writeVarInt(/*major=*/1);
  writer.writeVarInt(/*minor=*/2);
}

std::unique_ptr<mlir::DialectVersion>
MyDialectBytecodeInterface::readVersion(
    mlir::DialectBytecodeReader &reader) const {
  auto ver = std::make_unique<MyDialectVersion>();
  if (failed(reader.readVarInt(ver->major)) ||
      failed(reader.readVarInt(ver->minor)))
    return nullptr;
  return ver;
}

// In readType, check version to handle old encodings:
mlir::Type MyDialectBytecodeInterface::readType(
    mlir::DialectBytecodeReader &reader) const {
  auto verResult = reader.getDialectVersion<MyDialect>();
  uint64_t major = 0;
  if (succeeded(verResult)) {
    auto *ver = llvm::cast<MyDialectVersion>(*verResult);
    major = ver->major;
  }
  // Handle encoding differences between major versions...
}
```

---

## 136.6 External Resources

### The Resource Problem

Dense attribute values for ML model weights can be enormous — a `dense<...>` attribute for a 512×512×3 float tensor embeds 786,432 numbers in the textual format and a non-trivial encoding in the bytecode. MLIR addresses this with *external resources*: binary blobs stored separately from the op tree, referenced by name.

### DenseResourceElementsAttr

[`DenseResourceElementsAttr`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/BuiltinAttributes.h#L699) is the builtin mechanism:

```cpp
#include "mlir/IR/BuiltinAttributes.h"

// Create a weight tensor from a raw buffer:
llvm::SmallVector<float> weights(1024 * 512, 0.0f);
// ... populate weights ...

// Create an AsmResourceBlob that references the data
mlir::AsmResourceBlob blob = mlir::HeapAsmResourceBlob::allocate(
    llvm::ArrayRef(weights.data(), weights.size()),
    alignof(float), /*dataIsMutable=*/false);

auto shapedTy = mlir::RankedTensorType::get({1024, 512}, builder.getF32Type());
auto weightAttr = mlir::DenseResourceElementsAttr::get(
    shapedTy, "model_weights_layer0", std::move(blob));

// Attach to an op:
op->setAttr("weight", weightAttr);
```

In the textual format, `DenseResourceElementsAttr` prints as:
```mlir
{weight = dense_resource<model_weights_layer0> : tensor<1024x512xf32>}
```

The actual blob data is printed in a `{...}` resource section at the end of the module:

```mlir
{-#
  dialect_resources: {
    builtin: {
      model_weights_layer0: "0x..." // hex-encoded blob
    }
  }
#-}
```

### Resource Blobs in Bytecode

In the bytecode format, resource data lives in the `kResource` section (section ID 5) with an offset table in `kResourceOffset` (section ID 6). Blobs are stored with their required alignment and referenced by string ID from the `kString` section. The bytecode writer handles resource serialization automatically when `DenseResourceElementsAttr` is used; no special API calls are needed.

To prevent resource data from being emitted (for size-checking or testing):

```cpp
mlir::BytecodeWriterConfig config;
config.setElideResourceDataFlag(true);  // omit blob bytes, keep references
mlir::writeBytecodeToFile(module.get(), os, config);
```

The corresponding mlir-opt flag is `--elide-resource-data-from-bytecode`.

### FallbackAsmResourceMap

When parsing a textual `.mlir` file that contains a resource section, or when round-tripping bytecode through the textual form, a `FallbackAsmResourceMap` captures unknown resources:

```cpp
mlir::FallbackAsmResourceMap fallbackResources;
mlir::ParserConfig config(&ctx, /*verifyAfterParse=*/true, &fallbackResources);

auto module = mlir::parseSourceFile<mlir::ModuleOp>("model.mlir", config);

// Re-attach the captured resources when writing bytecode:
mlir::BytecodeWriterConfig writerConfig(fallbackResources);
mlir::writeBytecodeToFile(module.get(), os, writerConfig);
```

This pattern ensures resource blobs survive round-trips through the textual format without being dropped.

---

## 136.7 Translation: mlir-translate

### The Translation Framework

`mlir-translate` is MLIR's bidirectional bridge to non-MLIR IRs. It is distinct from `mlir-opt` (which runs passes within MLIR) in that it crosses IR boundaries: reading LLVM IR `.ll` files, emitting SPIR-V binaries, and so on. The framework is defined in [`mlir/include/mlir/Tools/mlir-translate/Translation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/mlir-translate/Translation.h).

Translations are registered statically via `TranslateToMLIRRegistration` (external format → MLIR) and `TranslateFromMLIRRegistration` (MLIR → external format):

```cpp
#include "mlir/Tools/mlir-translate/Translation.h"

// Register a "to MLIR" translation:
static mlir::TranslateToMLIRRegistration importLLVM(
    "import-llvm",
    "Translate LLVMIR to MLIR",
    [](llvm::SourceMgr &sourceMgr, mlir::MLIRContext *ctx)
        -> mlir::OwningOpRef<mlir::Operation *> {
      // ... read sourceMgr, return MLIR module ...
    },
    [](mlir::DialectRegistry &registry) {
      registry.insert<mlir::LLVM::LLVMDialect>();
    });

// Register a "from MLIR" translation:
static mlir::TranslateFromMLIRRegistration exportLLVMIR(
    "mlir-to-llvmir",
    "Translate MLIR to LLVMIR",
    [](mlir::Operation *op, llvm::raw_ostream &os) -> mlir::LogicalResult {
      // ... lower op to llvm::Module, emit IR ...
      return mlir::success();
    },
    [](mlir::DialectRegistry &registry) {
      registry.insert<mlir::LLVM::LLVMDialect,
                      mlir::func::FuncDialect>();
    });
```

### Available Translations in LLVM 22

The built-in translations available in LLVM 22.1.x (verified with `mlir-translate --help`):

| Flag | Direction | Description |
|------|-----------|-------------|
| `--mlir-to-llvmir` | MLIR → LLVM IR | Lowers MLIR LLVM dialect to `llvm::Module`; emits `.ll` textual IR |
| `--import-llvm` | LLVM IR → MLIR | Reads `.ll` or `.bc` files; imports into MLIR LLVM dialect |
| `--mlir-to-cpp` | MLIR → C++ | Emits C++ source via EmitC dialect lowering |
| `--import-wasm` | WebAssembly → MLIR | Experimental WebAssembly import |

Note: `--import-spirv` and `--mlir-to-spirv` are available in downstream builds that include the SPIR-V translation library; they are not present in the standard Ubuntu LLVM 22 package. The full set of translations depends on which libraries are linked into the `mlir-translate` binary.

### A Complete Linalg → LLVM IR Pipeline

This pipeline demonstrates the typical journey from high-level MLIR dialects to LLVM IR for code generation:

```bash
#!/bin/bash
OPT=/usr/lib/llvm-22/bin/mlir-opt
TRANSLATE=/usr/lib/llvm-22/bin/mlir-translate

# Input: linalg.matmul on memrefs
cat > /tmp/matmul.mlir << 'EOF'
func.func @matmul(%A: memref<4x4xf32>, %B: memref<4x4xf32>,
                  %C: memref<4x4xf32>) {
  linalg.matmul ins(%A, %B : memref<4x4xf32>, memref<4x4xf32>)
                outs(%C : memref<4x4xf32>)
  return
}
EOF

# Lower all the way to LLVM IR
$OPT /tmp/matmul.mlir \
  --linalg-generalize-named-ops \
  --convert-linalg-to-loops \
  --convert-scf-to-cf \
  --convert-cf-to-llvm \
  --convert-arith-to-llvm \
  --convert-memref-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts \
  -o /tmp/matmul_llvm.mlir

# Translate MLIR LLVM dialect → LLVM IR textual format
$TRANSLATE --mlir-to-llvmir /tmp/matmul_llvm.mlir -o /tmp/matmul.ll

# Optionally compile to native object:
/usr/lib/llvm-22/bin/llc /tmp/matmul.ll -o /tmp/matmul.s
```

### Implementing a Custom Translation

Out-of-tree tools frequently need custom translations (e.g., importing a hardware-specific format or exporting to a proprietary serialization):

```cpp
// my_translate_main.cpp
#include "mlir/Tools/mlir-translate/MlirTranslateMain.h"
#include "mlir/Tools/mlir-translate/Translation.h"
#include "MyDialect/IR/MyDialect.h"
#include "MyDialect/Translate/MyTranslation.h"

// Static registration fires before main():
static mlir::TranslateFromMLIRRegistration exportMyFormat(
    "export-my-format",
    "Export MyDialect to proprietary binary format",
    [](mlir::Operation *op, llvm::raw_ostream &os) {
      return my_dialect::exportToMyFormat(op, os);
    },
    [](mlir::DialectRegistry &registry) {
      registry.insert<my_dialect::MyDialect>();
    });

int main(int argc, char **argv) {
  mlir::registerAllTranslations();  // register builtins
  return mlir::mlirTranslateMain(argc, argv, "my-translate");
}
```

The `TranslateFromMLIRFunction` signature receives the top-level operation (typically a `ModuleOp`) and an output stream. The translation is responsible for calling `verifyAllDialectInterfaces` or doing its own verification if needed.

---

## 136.8 Custom Dialects and Serialization

### Registration Pattern

The canonical pattern for an out-of-tree dialect that supports bytecode serialization:

```cpp
// MyDialect.cpp

#include "mlir/Bytecode/BytecodeImplementation.h"

namespace {
class MyDialectBytecodeInterface : public mlir::BytecodeDialectInterface {
public:
  using BytecodeDialectInterface::BytecodeDialectInterface;

  mlir::Attribute readAttribute(
      mlir::DialectBytecodeReader &reader) const override {
    uint64_t tag;
    if (failed(reader.readVarInt(tag))) return {};
    switch (tag) {
    case 0: return readMySpecialAttr(reader);
    default:
      reader.emitError() << "unknown MyDialect attribute tag: " << tag;
      return {};
    }
  }

  mlir::LogicalResult writeAttribute(
      mlir::Attribute attr,
      mlir::DialectBytecodeWriter &writer) const override {
    return llvm::TypeSwitch<mlir::Attribute, mlir::LogicalResult>(attr)
      .Case<MySpecialAttr>([&](MySpecialAttr a) {
        writer.writeVarInt(0);
        return writeMySpecialAttr(a, writer);
      })
      .Default([](mlir::Attribute) { return mlir::failure(); });
  }

  // Type read/write omitted for brevity — same pattern as above
};
} // namespace

void MyDialect::initialize() {
  addOperations<
#define GET_OP_LIST
#include "MyDialectOps.cpp.inc"
  >();
  addTypes<MyVectorType, MyStructType>();
  addAttributes<MySpecialAttr>();
  addInterface<MyDialectBytecodeInterface>(*this);
}
```

### Blob Writing for Custom Types

For types that contain large data (e.g., sparse index buffers), the writer provides `writeOwnedBlob`:

```cpp
mlir::LogicalResult writeSparseData(MySparseAttr attr,
                                     mlir::DialectBytecodeWriter &writer) {
  auto indices = attr.getIndices();  // ArrayRef<int64_t>
  auto values  = attr.getValues();   // ArrayRef<float>

  writer.writeVarInt(indices.size());
  writer.writeOwnedBlob(
      llvm::ArrayRef(reinterpret_cast<const char *>(indices.data()),
                     indices.size() * sizeof(int64_t)));
  writer.writeOwnedBlob(
      llvm::ArrayRef(reinterpret_cast<const char *>(values.data()),
                     values.size() * sizeof(float)));
  return mlir::success();
}
```

`writeOwnedBlob` stores the raw bytes in the resource section with proper alignment tracking; the reader recovers them with `readBlob` or `readOptionalBlob`.

### FileCheck-Based Bytecode Tests

For regression testing custom bytecode encoding, combine `mlir-opt --emit-bytecode` with hexdump verification via `xxd` and FileCheck:

```bash
// RUN: mlir-opt --emit-bytecode %s | xxd | FileCheck %s --check-prefix=HEX
// RUN: mlir-opt --emit-bytecode %s | mlir-opt | FileCheck %s

// HEX: 4d4c ef52
// HEX: {{[0-9a-f ]+}}

func.func @test(%arg0: !my_dialect.my_vector<4xi32>) -> i32 {
  // ...
}
```

The round-trip test (emit bytecode then re-parse) is more valuable in practice than hex-level testing; it verifies the read path without brittle byte-position assertions.

---

## 136.9 MLIR in Production Pipelines

### Bytecode as Compiler Artifacts

In production ML compilers (IREE, OpenXLA, torch-mlir), `.mlirbc` files serve as compilation cache entries and intermediate artifacts:

```
source model (.pb, .onnx, .mlir)
     │
     ▼ import pass
high-level MLIR (.mlirbc)     ← cache checkpoint A
     │
     ▼ optimization passes
optimized MLIR (.mlirbc)      ← cache checkpoint B
     │
     ▼ lowering pipeline
LLVM dialect MLIR (.mlirbc)   ← cache checkpoint C
     │
     ▼ mlir-translate
LLVM IR (.ll / .bc)
     │
     ▼ llc / lld
native binary
```

Caching at bytecode checkpoints provides significant build-time speedups: subsequent compilations that start from an unchanged checkpoint skip all passes above it. The deterministic nature of MLIR bytecode (same IR → same bytes, given a fixed MLIR version) makes content-addressed caching (comparing file hashes) reliable.

### Versioning Concerns

Bytecode files are tightly coupled to the MLIR version that produced them. Key risks:

- **Schema evolution**: if a dialect's `BytecodeDialectInterface` changes (new discriminator tags, reordered fields), old `.mlirbc` files may fail to parse. This is why dialect versioning (Section 136.5) matters: the `readVersion`/`writeVersion` hooks allow the reader to dispatch to legacy read paths.
- **Op property changes**: if an op's `Properties` struct changes (renamed fields, different types), files produced at an older ODS layout may mis-parse. MLIR mitigates this through the `kNativePropertiesEncoding` version gate.
- **Recommended practice**: store the MLIR producer version string (embedded in the file header) alongside cache entries and invalidate when the compiler version changes.

### Multi-Module Test Files

`mlir-opt --split-input-file` processes `.mlir` test files containing multiple modules separated by the `// -----` sentinel. Each segment is parsed and processed independently:

```mlir
// First test: verify addi folding
// RUN: mlir-opt --canonicalize --split-input-file %s | FileCheck %s
// CHECK: return %arg0
func.func @fold_add_zero(%x: i32) -> i32 {
  %c0 = arith.constant 0 : i32
  %r = arith.addi %x, %c0 : i32
  return %r : i32
}

// -----

// Second test: verify muli folding
// CHECK: return %arg0
func.func @fold_mul_one(%x: i32) -> i32 {
  %c1 = arith.constant 1 : i32
  %r = arith.muli %x, %c1 : i32
  return %r : i32
}
```

`--split-input-file` is supported by all the MLIR tools (`mlir-opt`, `mlir-translate`, `mlir-runner`).

### The MLIR Language Server

`mlir-lsp-server` provides Language Server Protocol support for `.mlir` files: diagnostic squiggles, type hover, jump-to-definition for symbols, and inlay type hints. Run it as:

```bash
# Start the LSP server for VS Code / Neovim
/usr/lib/llvm-22/bin/mlir-lsp-server

# With include paths for dialect headers:
/usr/lib/llvm-22/bin/mlir-lsp-server \
  --mlir-lsp-server-params='{"include_dirs": ["/usr/lib/llvm-22/include"]}'
```

Configure in VS Code via the `vscode-mlir` extension by pointing `mlir.server_path` to the binary. In Neovim, configure `nvim-lspconfig` with `mlir_lsp_server`.

`mlir-pdll-lsp-server` provides the equivalent for `.pdll` files, with knowledge of ODS-generated op declarations when the build directory is on its include path.

---

## Chapter Summary

- **The textual format** provides exact round-trip serialization with human-readable syntax: SSA names, typed attributes, and dialect-extensible type/op printing; `parseSourceFile` auto-detects text vs bytecode.

- **The bytecode format** uses a 4-byte magic (`ML\xefR`) followed by 9 section types: string table, dialect info, attribute/type encoding, IR tree, resource blobs, resource offsets, dialect versions, and properties. Binary encoding is 3–5× smaller and 8–12× faster to parse than textual.

- **Bytecode version 6** is current in LLVM 22; versions 0–6 are all readable by the LLVM 22 reader. `BytecodeWriterConfig::setDesiredBytecodeVersion` targets older readers at the cost of potentially rejecting newer IR features.

- **`writeBytecodeToFile` / `parseSourceFile`** are the primary write/read API entry points; `BytecodeReader` provides lazy-loading for large modules where only a subset of functions is needed.

- **`BytecodeDialectInterface`** allows dialects to provide compact binary encodings for their types and attributes via `readType`/`writeType`/`readAttribute`/`writeAttribute`; failure falls back to textual-in-binary encoding.

- **`DenseResourceElementsAttr`** and `AsmResourceBlob` store large binary blobs (model weights) in the bytecode resource section with proper alignment, avoiding their embedding in the attribute tree.

- **`mlir-translate`** bridges MLIR to external IRs: `--mlir-to-llvmir` lowers MLIR LLVM dialect to LLVM IR text, `--import-llvm` imports `.ll` files. Custom translations register via `TranslateFromMLIRRegistration` / `TranslateToMLIRRegistration`.

- **Production patterns**: use `.mlirbc` files as cached pipeline checkpoints; rely on dialect versioning for forward-compatible schema evolution; use `--split-input-file` for multi-test `.mlir` files; deploy `mlir-lsp-server` and `mlir-pdll-lsp-server` for IDE integration.

- Key source locations: [`mlir/include/mlir/Bytecode/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Bytecode/), [`mlir/include/mlir/Parser/Parser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Parser/Parser.h), [`mlir/include/mlir/Tools/mlir-translate/Translation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/mlir-translate/Translation.h), [`mlir/include/mlir/IR/BuiltinAttributes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/BuiltinAttributes.h#L699).
