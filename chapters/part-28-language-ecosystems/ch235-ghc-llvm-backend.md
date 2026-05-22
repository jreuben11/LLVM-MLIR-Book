# Chapter 235 — GHC's LLVM Backend: Haskell to Native via LLVM

*Part XXVIII — Language-Specific Compilation Ecosystems*

The Glasgow Haskell Compiler (GHC) occupies a unique position in the LLVM ecosystem: it is the most elaborate example of a typed functional language compiling through LLVM IR, and the only production compiler that maintains rich type-theoretic structure (System FC coercion proofs) through multiple intermediate representations before erasing them at the LLVM boundary. GHC's `-fllvm` backend routes code through the LLVM optimizer instead of GHC's own native code generator (NCG), gaining access to LLVM's vectorizer, instruction combiner, and target-specific lowering—at the cost of longer compilation times. This chapter traces the full pipeline from Haskell source through GHC Core, STG, Cmm, and into LLVM IR, with particular attention to GHC's custom `ghccc` calling convention, the virtual register mapping, and the approach to precise garbage collection without LLVM's statepoint infrastructure.

## 235.1 The GHC Compilation Pipeline

GHC transforms Haskell through a sequence of progressively lower-level intermediate representations:

```
Haskell source (.hs)
    │
    ▼  Parsing + renaming
GHC Core (System FC)
    │  Core-to-Core optimizer (inlining, specialisation, case-of-case,
    │  worker-wrapper, strictness analysis, let-floating, CSE, rules)
    ▼
Prepared Core (for code generation)
    │
    ▼  CoreToStg (GHC.CoreToStg)
STG (Spineless Tagless G-machine)
    │
    ▼  StgToCmm (GHC.StgToCmm.codeGen)
Cmm (C--)
    │  Cmm passes: common block elim, proc-point analysis, SRT analysis,
    │  stack layout, info-table sinking
    ▼  (when -fllvm is active)
LLVM IR text/bitcode
    │
    ▼  opt (LLVM optimizer) + llc (LLVM backend)
Native object file (.o)
```

The key source files for the LLVM backend are all under `compiler/GHC/CmmToLlvm/`:

| File | Role |
|------|------|
| `CmmToLlvm.hs` | Entry point: `llvmCodeGen` |
| `Base.hs` | `LlvmM` monad, `LlvmCgConfig` |
| `CodeGen.hs` | Instruction-level codegen |
| `Regs.hs` | Virtual register → LLVM variable mapping |
| `Ppr.hs` | LLVM IR pretty-printer |
| `Data.hs` | LLVM IR data structures |
| `Version.hs` | LLVM version detection |
| `Version/Bounds.hs.in` | `supportedLlvmVersionLowerBound/UpperBound` |

## 235.2 GHC Core and System FC

GHC Core is a typed intermediate language based on System FC—System F extended with *coercion proofs* that witness type equalities. Core serves as both the optimization IR and a type-theoretic foundation; every Core-to-Core transformation is type-preserving and checked by *Core Lint*, GHC's internal type-checker for Core terms.

### Coercion Proofs

A `newtype` declaration creates a zero-cost type distinction enforced by coercion:

```haskell
newtype Age = Age Int
-- GHC Core represents: coercion Age_Co :: Age ~R# Int
--   (R = representational equality: same representation, different types)

-- newtype coerce in Core:
cast (x :: Age) (Age_Co :: Age ~R# Int) :: Int
```

The `cast` term applies a coercion proof `Age_Co`, allowing the optimizer to eliminate the coercion (no-op at runtime) while the type system catches misuse. `GeneralizedNewTypeDeriving` uses this mechanism extensively; `coerce :: Coercible a b => a -> b` is the user-facing API.

### Core Lint

`Core Lint` (`compiler/GHC/Core/Lint.hs`) typechecks the Core IR after each optimization pass. A transformation that introduces a type error is caught immediately. This is the safety net that allows GHC to apply aggressive transformations (worker-wrapper, let-floating, case-of-case) without risk of producing ill-typed machine code.

### Key Core-to-Core Passes

| Pass | Effect |
|------|--------|
| Inliner | Substitutes definitions at call sites based on size/benefit heuristics |
| Specialiser | Generates monomorphic specializations of polymorphic functions |
| Worker-Wrapper | Splits a function into a "wrapper" (handles unboxing) and "worker" (operates on unboxed values) |
| Case-of-Case | `case (case x of A → e1; B → e2) of ...` → push outer case inside |
| Strictness Analyser | Determines which arguments are always evaluated; enables unboxing |
| FloatOut / FloatIn | Moves let-bindings to minimize re-evaluation |

## 235.3 The STG Machine

The Spineless Tagless G-machine (Peyton Jones, JFP 1992) is GHC's operational model for lazy evaluation. "Spineless" means there is no spine stack for return frames; "tagless" means closures do not carry a tag distinguishing evaluated from unevaluated—instead, every closure's entry code handles both cases.

### STG Constructs

`StgBinding` in `compiler/GHC/Stg/Syntax.hs` defines the STG language:

```
StgTopBinding
  = StgTopLifted StgBinding   -- top-level closure
  | StgTopStringLit Id ByteString

StgBinding
  = StgNonRec Id StgRhs
  | StgRec [(Id, StgRhs)]

StgRhs
  = StgRhsClosure CostCentreStack StgBinderInfo [Id] StgExpr
      --          cost centre     free vars      args   body
  | StgRhsCon CostCentreStack DataCon [StgArg]
      -- constructor application (evaluated, no thunk needed)

StgExpr
  = StgApp Id [StgArg]         -- function application
  | StgLit Literal             -- literal constant
  | StgConApp DataCon [StgArg] -- saturated constructor
  | StgOpApp StgOp [StgArg]    -- primop application
  | StgCase StgExpr Id AltType [StgAlt]  -- evaluation + case analysis
  | StgLet (BinderP 'Lifted) StgBinding StgExpr
  | StgLetNoEscape ...
  | StgTick ...
```

### Heap Allocation and Entry Conventions

Every closure is a heap object with an **info pointer** as its first word. The info pointer points to an **info table** that precedes the closure's entry code:

```
                  ┌─────────────────────────────┐
info table        │ SRT pointer (static roots)  │ ← GC static reference table
(read-only data)  │ Layout info (ptrs bitmap)   │
                  │ Closure type                │
                  │ Entry code                  │ ← info pointer points here
                  ├─────────────────────────────┤
heap object       │ info pointer                │ ← closure pointer
                  │ free variable 1             │
                  │ free variable 2             │
                  │ ...                         │
                  └─────────────────────────────┘
```

**Evaluating a thunk**: dereference the closure pointer to get the info pointer, call the entry code. The entry code either returns a value (if already evaluated) or computes and overwrites itself (update frame on the stack, then evaluate).

**Black-holing**: when a thunk begins evaluation, it overwrites its info pointer with a black-hole info table. If evaluation loops, re-entering the black hole throws a `<<loop>>` exception.

## 235.4 Cmm (C--)

Cmm is GHC's low-level imperative IR, analogous to LLVM IR but with explicit stack manipulation and info table annotations. It is defined in `compiler/GHC/Cmm/`.

### GenCmmDecl and CmmProc

```haskell
-- compiler/GHC/Cmm.hs
data GenCmmDecl d h g
  = CmmProc h CLabel [(CLabel, [Unique])] g
    -- ^ A procedure: info table header, entry label,
    --   internal labels (for proc-point splitting), control-flow graph
  | CmmData Section [d]
    -- ^ Static data section

type CmmGroup    = GenCmmGroup CmmStatics CmmTopInfo CmmGraph
type RawCmmGroup = GenCmmGroup RawCmmStatics (LabelMap RawCmmStatics) CmmGraph
```

A `CmmProc` bundles an info table (`h`) with the procedure body (`g`). The info table is emitted immediately before the entry point in the object file—this is how GHC's runtime traverses the stack: given a return address, subtract `sizeof(info_table)` to reach the info table.

### Stack Layout in Cmm

Cmm uses an explicit stack pointer (`Sp`) rather than a hardware frame pointer. Each procedure declares the maximum stack frame size; the Cmm code generator inserts stack overflow checks at function entry (`SpLim` is the stack limit register).

A typical Cmm procedure entry:

```
entry_foo:
    // Stack overflow check
    if (Sp - 24 < SpLim) { call stg_gc_fun; }
    // Save closure pointer (R1) for GC
    I64[Sp - 8] = R1;
    Sp = Sp - 24;
    // Body ...
    jump stg_upd_frame [R1];  // tail call to update frame
```

### SRT (Static Reference Table)

The SRT is a per-closure list of statically-allocated CAFs (Constant Applicative Forms—top-level thunks) that the closure can reach. The GC uses the SRT to find live CAFs and prevent them from being collected while any closure referencing them is live. SRT analysis runs as a Cmm pass (`compiler/GHC/Cmm/CmmBuildInfoTables.hs`) producing `CmmGroupSRTs`.

## 235.5 LlvmCodeGen Module

The LLVM backend's entry point is `llvmCodeGen` in `compiler/GHC/CmmToLlvm.hs`:

```haskell
llvmCodeGen :: Logger
            -> LlvmCgConfig
            -> Handle          -- output handle for LLVM IR text
            -> DUniqSupply     -- deterministic unique supply
            -> CgStream RawCmmGroup a
            -> IO a
```

The function processes `RawCmmGroup` values as they stream out of the Cmm pipeline, writing LLVM IR to the `Handle` incrementally (GHC avoids materializing the full IR in memory).

### LlvmM Monad

```haskell
-- compiler/GHC/CmmToLlvm/Base.hs
type LlvmM a = ReaderT LlvmEnv IO a

data LlvmEnv = LlvmEnv
  { envConfig   :: LlvmCgConfig
  , envVersion  :: LlvmVersion
  , envOutput   :: BufHandle
  , envUniq     :: DUniqSupply
  , envAliases  :: IORef (UniqSet LMString)  -- emitted type aliases
  , envGlobs    :: IORef (UniqSet LMString)  -- emitted globals
  }
```

`LlvmM` provides access to the LLVM version (for version-specific IR generation), a buffered output handle, and mutable state for deduplication of type aliases and global variable declarations.

### LlvmType

```haskell
-- compiler/GHC/Llvm/Types.hs
data LlvmType
  = LMInt Int              -- integer of given width: LMInt 64 = i64
  | LMFloat                -- f32
  | LMDouble               -- f64
  | LMFloat80              -- x86 extended precision
  | LMFloat128             -- f128
  | LMPointer LlvmType     -- pointer
  | LMArray Int LlvmType   -- [N x T]
  | LMVector Int LlvmType  -- <N x T>
  | LMLabel                -- label (for branch targets)
  | LMVoid
  | LMStruct [LlvmType]    -- packed struct {T1, T2, ...}
  | LMStructU [LlvmType]   -- unpacked struct
  | LMAlias LlvmAlias      -- %name = type T
  | LMMetadata
  | LMFunction LlvmFunctionDecl
```

GHC maps all Haskell values to word-sized integers (`LMInt 64` on 64-bit) or pointers (`LMPointer (LMInt 8)` — opaque `i8*` → `ptr` in LLVM 15+). Floating-point values use `LMFloat`/`LMDouble`. SIMD types use `LMVector`.

## 235.6 The `ghccc` Calling Convention

GHC's most distinctive LLVM feature is the `ghccc` calling convention (LLVM calling convention 10). It reflects the STG machine's register-passing model: function arguments and continuation results are passed in a fixed set of virtual STG registers rather than through the system ABI.

```haskell
-- compiler/GHC/Llvm/Types.hs
data LlvmCallConvention
  = CC_Ccc           -- C calling convention
  | CC_Fastcc        -- LLVM fastcc
  | CC_Coldcc        -- LLVM coldcc
  | CC_Ghc           -- GHC calling convention (ghccc)
  | CC_Ncc Int       -- Numeric: CC_Ncc 10 == ghccc
  | CC_X86_Stdcc
  deriving (Eq)

ppLlvmCallConvention :: LlvmCallConvention -> SDoc
ppLlvmCallConvention CC_Ghc = text "ghccc"
```

### Virtual Register Mapping

`lmGlobalReg` (`compiler/GHC/CmmToLlvm/Regs.hs`) maps each STG virtual register to an LLVM global variable:

| STG register | LLVM variable | Type | Role |
|-------------|---------------|------|------|
| `VanillaReg 1` | `R1` | `i64` | First heap pointer / return value |
| `VanillaReg 2..10` | `R2..R10` | `i64` | Additional arguments |
| `Sp` | `Sp` | `i64*` | Stack pointer |
| `SpLim` | `SpLim` | `i64` | Stack limit (overflow check) |
| `Hp` | `Hp` | `i64*` | Heap pointer |
| `HpLim` | `HpLim` | `i64` | Heap limit (allocation check) |
| `BaseReg` | `Base` | `i64*` | Pointer to thread-state object (`Capability`) |
| `FloatReg 1..6` | `F1..F6` | `float` | Float arguments/results |
| `DoubleReg 1..6` | `D1..D6` | `double` | Double arguments/results |
| `XmmReg 1..6` | `XMM1..XMM6` | `<4 x float>` | SIMD |

**`alwaysLive`**: the registers `[BaseReg, Sp, Hp, SpLim, HpLim, node]` are always considered live and are never dead-code-eliminated. This prevents LLVM from optimizing away the stack/heap pointer updates that the GC depends on.

```haskell
alwaysLive :: Platform -> [GlobalRegUse]
alwaysLive platform =
  [ GlobalRegUse r (globalRegSpillType platform r)
  | r <- [BaseReg, Sp, Hp, SpLim, HpLim, node] ]
```

### Tail Calls

GHC makes heavy use of tail calls—nearly every STG function call is a tail call. Under `ghccc`, LLVM emits `musttail call ghccc` instructions, which compile to direct jumps on x86-64. The STG machine's continuation-passing style means the call stack is GHC's explicit `Sp` stack, not the hardware call stack.

## 235.7 GC Integration

GHC does not use LLVM's statepoint/gc.relocate infrastructure (Chapter 58). Instead, GHC manages precise GC through a combination of:

1. **Info tables**: every closure's GC layout is recorded in its info table (pointer bitmap and pointer count)
2. **Stack traversal**: the GC walks the Sp stack by following return address → info table → pointer bitmap at each frame
3. **Heap traversal**: the GC walks the heap by following the closure pointer → info table for each object

### stgTBAA: Alias Analysis Metadata

`stgTBAA` (`compiler/GHC/CmmToLlvm/Regs.hs`) defines TBAA (Type-Based Alias Analysis) metadata that allows LLVM to reason about heap vs. stack accesses:

```haskell
stgTBAA :: [(Unique, LMString, Maybe Unique)]
stgTBAA =
  [ (rootN,  fsLit "root",   Nothing)
  , (topN,   fsLit "top",    Just rootN)
  , (stackN, fsLit "stack",  Just topN)
  , (heapN,  fsLit "heap",   Just topN)
  , (rxN,    fsLit "rx",     Just topN)    -- return registers
  ]
```

With this hierarchy: heap loads/stores (`heapN`) cannot alias stack loads/stores (`stackN`). This enables LLVM to hoist heap reads out of loops and sink heap writes, a significant optimization for functional programs that allocate heavily.

### Interaction with LLVM GC Features

GHC deliberately avoids LLVM's `@llvm.gcroot`, `@llvm.experimental.stackmap`, and `RewriteStatepointsForGC`. These mechanisms would require LLVM to be aware of GHC's heap layout and calling convention, creating a tight coupling. Instead, GHC's GC is entirely managed by GHC's runtime (`rts/`) and only interacts with the LLVM backend through:
- Info tables emitted as `.data` adjacent to code
- The `alwaysLive` register set preventing their optimization-away
- TBAA metadata for alias analysis

## 235.8 `-fllvm` vs. the Native Code Generator

GHC ships two backends: the Native Code Generator (NCG, `compiler/GHC/CmmToAsm/`) and the LLVM backend (`-fllvm`).

### When `-fllvm` Wins

| Workload | NCG | `-fllvm` | Notes |
|----------|-----|----------|-------|
| Numeric / SIMD | Moderate | **Better** | LLVM auto-vectorizer; `llvm.fma.f64`; AVX-512 |
| Heavy pointer chasing | Fast | Similar | LLVM loop opts less effective on pointer-based code |
| Template Haskell / compile speed | **Faster** | 2–3× slower | LLVM overhead per TH splice |
| Code size | Similar | **Smaller** | LLVM's ICF and dead-stripping |
| Profiling builds (`-prof`) | OK | **Better** | LLVM's cost-centre instrumentation |

Typical observation: `-fllvm` produces 10–30% faster runtime code for arithmetic-heavy programs (numerical simulation, parsing, compression) at 2–3× longer compile time.

### Supported LLVM Versions

GHC 9.x (the current series) supports LLVM 13 through 22:

```haskell
-- compiler/GHC/CmmToLlvm/Version/Bounds.hs.in (generated by configure)
supportedLlvmVersionLowerBound :: LlvmVersion
supportedLlvmVersionLowerBound = LlvmVersion (13 NE.:| [])

supportedLlvmVersionUpperBound :: LlvmVersion  -- exclusive upper bound
supportedLlvmVersionUpperBound = LlvmVersion (23 NE.:| [])
```

GHC detects the LLVM version at compilation time by running `llvm-config --version`. If the detected version is outside the supported range, GHC emits a warning and attempts codegen anyway (falling back to NCG on hard failure).

### Using `-fllvm`

```bash
# Single file
ghc -fllvm -O2 Main.hs

# Via cabal
cabal build --ghc-options="-fllvm -O2"

# Check which LLVM version GHC found
ghc -fllvm -v 2>&1 | grep "llvm version"

# Force a specific llc / opt path
ghc -fllvm -pgmlo /usr/lib/llvm-22/bin/opt \
            -pgmlc /usr/lib/llvm-22/bin/llc \
            -O2 Main.hs
```

### Debugging the LLVM Backend

```bash
# Dump LLVM IR before optimization
ghc -fllvm -O2 -ddump-llvm Main.hs

# Keep intermediate files
ghc -fllvm -O2 -keep-llvm-file Main.hs
# Produces Main.ll (LLVM IR text)
```

## Chapter Summary

- GHC compiles Haskell through **GHC Core (System FC)** → **STG** → **Cmm** → **LLVM IR**; types are maintained through Core, erased at Cmm
- **System FC coercion proofs** make newtype coercions type-safe zero-cost operations; **Core Lint** typechecks Core after every optimization pass
- **STG** models lazy evaluation with thunks, closures, and info tables; the entry convention is continuation-passing via the `Sp` stack, not the hardware call stack
- **Cmm** is an explicit-stack IR with `CmmProc` bundling info tables with procedure bodies; SRT analysis builds per-closure static root tables for the GC
- **`llvmCodeGen`** processes streaming `RawCmmGroup`s via the `LlvmM` monad; `LlvmType` covers integer, float, pointer, SIMD, struct, and function types
- **`ghccc`** (LLVM CC 10) maps STG virtual registers (`R1–R10`, `Sp`, `Hp`, `SpLim`, `HpLim`) to LLVM global variables; `alwaysLive` prevents their elimination; `musttail` enables jump-based continuation calls
- **GC integration** uses info tables + stack/heap traversal rather than LLVM statepoints; **`stgTBAA`** metadata separates heap from stack for LLVM alias analysis
- **`-fllvm`** supports LLVM 13–22; wins on SIMD-heavy and numeric workloads; costs 2–3× longer compile time; activated via `-fllvm` with optional `-pgmlo`/`-pgmlc` to select specific LLVM tools

**Cross-references**: [Chapter 15 — Type Theory in Practice](../part-03-type-theory/ch15-type-theory-in-practice.md) | [Chapter 193 — Julia: Type-Inference-Driven LLVM Specialization](ch193-julia-llvm-specialization.md) | [Chapter 228 — Virtual Machine Design](../part-16-jit-sanitizers/ch228-virtual-machine-design.md) | [Chapter 58 — Language Runtime Concerns](../part-09-frontend-authoring/ch58-language-runtime-concerns.md)

**References**:
- Peyton Jones. "Implementing Lazy Functional Languages on Stock Hardware: The STG Machine." Journal of Functional Programming 2(2), 1992
- Peyton Jones et al. "A Transformation-Based Optimiser for Haskell." Science of Computer Programming 32(1-3), 1998 (Secrets of the GHC Inliner)
- Sulzmann et al. "System F with Type Equality Coercions." TLDI 2007 (System FC)
- [GHC Commentary: CmmToLlvm](https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/CmmToLlvm.hs)
- [GHC User's Guide: LLVM Backend](https://ghc.gitlab.haskell.org/ghc/doc/users_guide/codegens.html#llvm-code-generator-fllvm)
