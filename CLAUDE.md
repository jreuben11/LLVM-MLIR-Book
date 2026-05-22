# LLVM, Clang & MLIR — The Expert Programming Reference
## CLAUDE.md — Project Instructions for AI Agents

---

## Project Overview

This is a ~3,050-page expert reference book covering the LLVM compiler ecosystem. It spans 239 chapters and 10 appendices across 31 parts. The book targets **LLVM 22.1.x / Clang 22 / MLIR in-tree / OpenXLA / ClangIR** as of April 2026.

The full outline is in `LLVM-MLIR-programming-book-plan.md`. Progress is tracked in `TODO_checklist.md` — always update it after completing a chapter.

---

## Directory Structure

```
LLVM-MLIR-Book/
├── CLAUDE.md
├── LLVM-MLIR-programming-book-plan.md
├── TODO_checklist.md
├── chapters/
│   ├── part-01-foundations/
│   ├── part-02-compiler-theory/
│   ├── part-03-type-theory/
│   ├── part-04-llvm-ir/
│   ├── part-05-clang-frontend/
│   ├── part-06-clang-codegen/
│   ├── part-07-clang-multilang/
│   ├── part-08-clangir/
│   ├── part-09-frontend-authoring/
│   ├── part-10-analysis-middle-end/
│   ├── part-11-polyhedral-theory/
│   ├── part-12-polly/
│   ├── part-13-lto-whole-program/
│   ├── part-14-backend/
│   ├── part-15-targets/
│   ├── part-16-jit-sanitizers/
│   ├── part-17-runtime-libs/
│   ├── part-18-flang/
│   ├── part-19-mlir-foundations/
│   ├── part-20-in-tree-dialects/
│   ├── part-21-mlir-transformations/
│   ├── part-22-xla-openxla/
│   ├── part-23-mlir-production/
│   ├── part-24-verified-compilation/
│   ├── part-25-operations-contribution/
│   ├── part-26-ecosystem-frontiers/
│   ├── part-27-mathematical-foundations/
│   ├── part-28-language-ecosystems/
│   ├── part-29-compiler-tooling/
│   ├── part-30-AI-first-PL-design/
│   └── part-31-frontier-ai-evolution/
└── appendices/
```

Chapter files are named: `chNN-kebab-case-title.md`

---

## Writing Standards

### Chapter structure (every chapter must have):
1. **Chapter title** (H1) and **part label** (italicized subtitle)
2. **Opening paragraph** — why this chapter matters and what the reader will understand
3. **Sections** (H2) with **subsections** (H3) as needed
4. **Code examples** — inline fenced blocks with language tags (`cpp`, `llvm`, `mlir`, `bash`, etc.)
5. **Cross-references** — explicit `[Chapter N — Title](../partXX/chNN-title.md)` links
6. **Reference links** — inline citations to canonical sources (papers, docs, source files)
7. **Chapter summary** — bullet list of key takeaways

### Code accuracy (CRITICAL):
- **Non-theoretical chapters**: All code must be verified against LLVM 22.1.x. Use:
  - `clang --version` → must show `22.x`
  - `llvm-config --version` → must show `22.x`
  - `clang -emit-llvm -S` to verify IR output
  - Source file references must use paths valid in the LLVM 22 monorepo
- **Theoretical chapters** (Parts II, III, XI, XXIV, XXVII): Verify against canonical literature cited per-chapter; no LLVM code verification required

### Page-length targets:
- Practical chapters: ~12 pages each (in markdown, ~600 words/page → ~7200 words)
- Theoretical chapters: ~20 pages each (~12000 words)
- Appendices: variable

### Style:
- Expert audience — assume C++ proficiency and compiler familiarity
- No padding or filler; every sentence earns its place
- Prefer showing actual LLVM source file paths and line-number ranges over abstract descriptions
- Inline reference links format: `[SourceFile](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/path/to/file.cpp#LNN)`
- Use tables for catalogues (instruction sets, attributes, calling conventions, etc.)
- Use diagrams in ASCII/text form where they clarify architecture

---

## Workflow Per Chapter

1. **Plan**: List the major sections and identify 2–3 key code examples to demonstrate
2. **Write**: Draft the full chapter in markdown
3. **Verify**: Run code examples through `clang`/`opt`/`mlir-opt` to confirm correctness
4. **Update** `TODO_checklist.md` — mark chapter done, note any improvements flagged
5. **Commit**: `git add chapters/... && git commit -m "ch<NN>: <title>"`

---

## Agent Parallelism Rules

- Chapters within the **same part** must be written **sequentially** (cross-references flow forward)
- Chapters in **different parts with no shared dependencies** can be written in **parallel**
- Safe parallel groups (no inter-group cross-refs at draft time):
  - Group A: Parts I, II, III (foundations — write first)
  - Group B: Part IV (LLVM IR) — after Group A
  - Group C: Parts V, VI, VII, VIII simultaneously — after Group B
  - Group D: Parts IX, X, XI simultaneously — after Group B
  - Group E: Parts XII, XIII, XIV, XV simultaneously — after Groups B+D
  - Group F: Parts XVI, XVII, XVIII simultaneously — after Group E
  - Group G: Parts XIX, XX, XXI simultaneously — after Group B
  - Group H: Parts XXII, XXIII — after Group G
  - Group I: Part XXIV — after all prior parts
  - Group J: Part XXV + Appendices — last
  - Group K: Parts XXVI, XXVII, XXVIII, XXIX simultaneously — after Group I
  - Group L: Parts XXX, XXXI — after Group K

---

## Toolchain Available (verified)

```
clang       : Ubuntu clang version 22.1.3
llvm-config : /usr/lib/llvm-22/bin/llvm-config
opt         : /usr/lib/llvm-22/bin/opt  (check path)
mlir-opt    : check /usr/lib/llvm-22/bin/ or build
python3     : 3.12.3
```

Prepend `/usr/lib/llvm-22/bin/` when running LLVM tools.

---

## Context / Compaction Continuity

Before context compaction, always:
1. Update `TODO_checklist.md` with current status and next chapter
2. Commit all written chapters
3. Add any quality improvement notes to the "Suggested Improvements" section of `TODO_checklist.md`

After resuming from compaction:
1. Read `TODO_checklist.md` first to orient
2. Continue from the next unchecked chapter

---

## References (canonical sources per theoretical part)

| Part | Sources |
|------|---------|
| II — Compiler Theory | Aho/Lam/Sethi/Ullman (Dragon Book 2e), Cooper-Torczon (EaC 3e), Appel (Modern Compiler Impl.), Muchnick |
| III — Type Theory | Pierce (TAPL, ATTAPL), Harper (PFPL), Mitchell, Reynolds |
| XI — Polyhedral Theory | Bondhugula PhD thesis, Feautrier scheduling papers, Verdoolaege ISL papers, Grosser et al. |
| XXIV — Verified Compilation | Leroy (CompCert), Zhao/Zdancewic/Nagarakatte (Vellvm), Lopes/Lee/Hur (Alive2) |
| XXVII — Mathematical Foundations for Compiler Engineers | Lean 4 / Coq / Isabelle/HOL / Agda reference manuals; Enderton (Mathematical Introduction to Logic); Klein et al. (seL4, SOSP 2009); Mac Lane (Categories for the Working Mathematician); Awodey (Category Theory); Winskel (Formal Semantics of Programming Languages) |
