# Chapter 176 — Contributing to LLVM

*Part XXV — Operations, Bindings, and Contribution*

LLVM is one of the most actively developed open-source compiler projects in the world: hundreds of contributors across industry (Apple, Google, Meta, ARM, Intel, AMD, Qualcomm, and many others) and academia commit tens of thousands of changes per year. Despite this scale, the project remains accessible to new contributors — the LLVM community is deliberately welcoming, the codebase is well-organized, and the review process is designed to transfer knowledge as well as approve changes. This chapter covers everything you need to make your first contribution: finding the right starting point, setting up a development workflow, understanding the code style requirements, navigating the review process, writing RFCs for significant changes, and understanding how the release process works. The emphasis is on practical guidance that gets you from "I want to contribute" to "my patch is merged."

---

## Table of Contents

- [176.1 The LLVM Community](#1761-the-llvm-community)
  - [Governance Structure](#governance-structure)
  - [Communication Channels](#communication-channels)
- [176.2 Getting Started](#1762-getting-started)
  - [Finding Work](#finding-work)
  - [Development Environment Setup](#development-environment-setup)
  - [Branch Workflow](#branch-workflow)
- [176.3 Code Style and Standards](#1763-code-style-and-standards)
  - [The LLVM Coding Standards](#the-llvm-coding-standards)
  - [clang-format](#clang-format)
  - [clang-tidy](#clang-tidy)
- [176.4 The Review Process](#1764-the-review-process)
  - [Opening a Pull Request](#opening-a-pull-request)
  - [CI System](#ci-system)
  - [Review Expectations](#review-expectations)
  - [Commit Access](#commit-access)
- [176.5 Writing an RFC](#1765-writing-an-rfc)
  - [When to Write an RFC](#when-to-write-an-rfc)
  - [RFC Format](#rfc-format)
- [Motivation](#motivation)
- [Proposed Design](#proposed-design)
- [Alternatives Considered](#alternatives-considered)
- [Impact on Existing Users](#impact-on-existing-users)
- [Implementation Plan](#implementation-plan)
  - [RFC for New MLIR Dialects](#rfc-for-new-mlir-dialects)
- [176.6 Release Process](#1766-release-process)
  - [Release Cycle](#release-cycle)
  - [Branching and the Branch Lifecycle](#branching-and-the-branch-lifecycle)
  - [Release Management](#release-management)
  - [What "Release" Means](#what-release-means)
  - [Deprecation Policy](#deprecation-policy)
- [Chapter Summary](#chapter-summary)

---

## 176.1 The LLVM Community

### Governance Structure

LLVM does not have a single benevolent dictator. Technical decisions are made by the community through discussion, typically on Discourse (discourse.llvm.org) or through the pull request review process on GitHub. The LLVM Foundation (a 501(c)(3) nonprofit) handles infrastructure, legal, and event coordination, but it does not control technical direction.

Sub-projects have de facto maintainers — individuals with deep expertise who are most active in reviewing and guiding changes. There is no formal maintainer list for most of LLVM, but `CODEOWNERS` files in the repository specify who should be notified about changes in particular directories:

```
# from llvm/CODEOWNERS (simplified)
/llvm/lib/Transforms/InstCombine/ @nikic @dtcxzyw
/llvm/lib/Analysis/               @nikic
/mlir/                            @mehdi_amini @ftynse @nicolasvasilache
/clang/lib/Sema/                  @erichkeane @AaronBallman
```

Reviewers listed in `CODEOWNERS` are automatically added to pull requests touching those files. This is the primary mechanism for finding the right reviewer.

### Communication Channels

**Discourse** (discourse.llvm.org): the main discussion forum. Organized into categories:
- `LLVM Project` → `Announce`: release announcements, major changes.
- `LLVM Project` → `Requests for Comments`: RFC proposals (§176.5).
- `Clang Frontend` → ...: Clang-specific discussions.
- `MLIR` → ...: MLIR-specific discussions.
- `Code Review`: discussion threads for in-flight reviews.

**GitHub Issues** (github.com/llvm/llvm-project/issues): bug reports and feature requests. The "good first issue" label marks issues suitable for new contributors.

**GitHub Pull Requests** (github.com/llvm/llvm-project/pulls): all code reviews happen here. Pre-commit CI runs automatically.

**LLVM Weekly Newsletter**: summarizes recent commits, RFCs, and community discussions; useful for staying up to date without following all Discourse threads.

**LLVM office hours**: irregular Zoom calls (announced on Discourse) where LLVM contributors answer questions. Particularly useful for newcomers who have questions that are easier to discuss interactively.

---

## 176.2 Getting Started

### Finding Work

Three good entry points:

**"Good first issue" labels**: GitHub Issues labeled `good first issue` are deliberately chosen for accessibility:
```
https://github.com/llvm/llvm-project/issues?q=is:open+label:"good+first+issue"
```

Typical good-first-issue categories:
- Adding a missing fold to InstCombine (write a test, add the rule, verify with alive-tv).
- Fixing a documentation error in the LangRef.
- Adding a missing diagnostic to Clang (e.g., a missing `-Wfoo` warning for a C++ pitfall).
- Improving a code quality issue (replacing raw pointer with `unique_ptr`, fixing a TODO comment).

**Stale TODO comments**: the LLVM codebase contains thousands of `TODO:` and `FIXME:` comments that represent known improvements. Many of these are approachable:
```bash
# Find TODOs in InstCombine
grep -r "TODO\|FIXME" llvm/lib/Transforms/InstCombine/ | head -20
```

**GSoC and LLF mentorship**: the LLVM Foundation participates in Google Summer of Code and runs its own mentorship program. Project ideas are posted before the GSoC application period; mentors are assigned to help students through the contribution process. Past projects include adding support for new architectures, improving MLIR dialects, and adding PGO-guided optimizations.

### Development Environment Setup

```bash
# 1. Fork the llvm/llvm-project repository on GitHub
# 2. Clone your fork
git clone https://github.com/YOUR_USERNAME/llvm-project.git
cd llvm-project

# 3. Add the upstream remote
git remote add upstream https://github.com/llvm/llvm-project.git

# 4. Configure build (debug build for development)
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_ENABLE_PROJECTS="clang;mlir" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DLLVM_USE_LINKER=lld \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++

# 5. Build relevant targets
ninja -C build opt clang mlir-opt

# 6. Run tests to confirm the build is healthy
ninja -C build check-llvm-transforms
ninja -C build check-clang
ninja -C build check-mlir
```

### Branch Workflow

```bash
# Always base your work on a fresh main
git fetch upstream
git checkout -b my-feature upstream/main

# Make your changes
# ... edit files ...

# Run relevant tests
ninja -C build check-llvm-transforms-instcombine

# Format code
clang-format -i -style=LLVM llvm/lib/Transforms/InstCombine/MyChange.cpp

# Commit
git commit -m "InstCombine: fold (x + C) - C to x

This adds a peephole rule that cancels a constant add/sub pair.
Verified with alive-tv."

# Push to your fork
git push origin my-feature
# Then open a Pull Request on GitHub
```

---

## 176.3 Code Style and Standards

### The LLVM Coding Standards

The authoritative reference: [llvm.org/docs/CodingStandards.html](https://llvm.org/docs/CodingStandards.html). Key points:

**Naming conventions**:
```cpp
// Variables and functions: camelCase
int myVariable;
void doSomething();

// Types and classes: PascalCase
class MyClass;
struct MyStruct;

// Constants and enums: PascalCase or SCREAMING_SNAKE_CASE for macros
const int MaxIterations = 100;
#define LLVM_LIKELY(x) __builtin_expect(!!(x), 1)

// Namespace names: lowercase
namespace llvm { ... }
```

**Include order** (alphabetical within each group, blank line between groups):
```cpp
// 1. The corresponding header (for .cpp files)
#include "llvm/Transforms/Scalar/MyPass.h"

// 2. LLVM headers
#include "llvm/Analysis/AliasAnalysis.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/PassManager.h"

// 3. Standard library headers
#include <memory>
#include <vector>
```

**No RTTI** (by default in LLVM): use `llvm::isa<>`, `llvm::cast<>`, `llvm::dyn_cast<>` instead of `dynamic_cast<>`. LLVM's own RTTI is faster (no vtable lookup; uses classof()).

**No exceptions**: LLVM is built with `-fno-exceptions`. Use `llvm::Error` and `llvm::Expected<T>` for error handling:
```cpp
llvm::Expected<int> safeDiv(int a, int b) {
  if (b == 0)
    return llvm::make_error<llvm::StringError>("division by zero",
                                               llvm::inconvertibleErrorCode());
  return a / b;
}
```

**Prefer LLVM types**:
- `llvm::StringRef` over `const std::string&` or `const char*`
- `llvm::ArrayRef<T>` over `const std::vector<T>&`
- `llvm::SmallVector<T, N>` over `std::vector<T>` for small collections
- `llvm::DenseMap<K, V>` over `std::unordered_map<K, V>` for frequent lookups

### clang-format

LLVM enforces formatting with `clang-format`:

```bash
# Format a single file
clang-format -i -style=LLVM my_file.cpp

# Format only changed lines (recommended for existing files)
git diff HEAD | clang-format-diff -p1 -i

# Check formatting without modifying
clang-format --dry-run --Werror -style=LLVM my_file.cpp
```

The `.clang-format` file in the LLVM root:
```yaml
BasedOnStyle: LLVM
ColumnLimit: 80
IndentWidth: 2
TabWidth: 2
UseTab: Never
BreakBeforeBraces: Attach
AllowShortFunctionsOnASingleLine: Empty
AlwaysBreakTemplateDeclarations: Yes
```

CI will fail if your code is not properly formatted. Format before every commit.

### clang-tidy

```bash
# Run clang-tidy on changed files
clang-tidy -p build/ \
  --config-file=.clang-tidy \
  llvm/lib/Transforms/InstCombine/MyChange.cpp

# Common checks LLVM uses:
# modernize-use-auto
# modernize-use-nullptr
# readability-identifier-naming
# llvm-prefer-isa-or-dyn-cast-in-conditionals
```

The `.clang-tidy` in the LLVM root enables project-specific checks including the `llvm-*` checks that enforce LLVM-specific idioms.

---

## 176.4 The Review Process

### Opening a Pull Request

```bash
# Push your branch to your fork
git push origin my-feature

# On GitHub: New pull request
# Base: llvm/llvm-project:main
# Head: YOUR_USERNAME/llvm-project:my-feature
```

PR title format: `[SubProject] Short description`. Examples:
- `[InstCombine] Fold (x + C) - C to x`
- `[Clang][Sema] Fix missing diagnostic for ambiguous conversion`
- `[MLIR][Affine] Add loop normalization to the affine loop utils`

PR body: explain what the change does, why it's correct, and any relevant context. Include:
- A brief description of the problem being solved.
- The approach taken.
- Test coverage (which test files were added/modified).
- For optimization passes: alive-tv output if applicable.

### CI System

LLVM's CI runs automatically on PRs:

**GitHub Actions** (`llvm/llvm-project`): runs a subset of tests on Linux x86-64 and macOS. Typically completes in 30–60 minutes.

**Buildkite** (builds.llvm.org): additional platforms (Windows, AArch64, RISC-V, various sanitizer builds). May take 1–3 hours.

**Pre-merge testing**: some sub-projects (Clang, MLIR) have stricter pre-merge requirements. Review the failing CI logs carefully — most failures are related to the change, but some are flaky tests or unrelated infrastructure issues.

If CI fails:
```bash
# Reproduce the failure locally
ninja -C build check-llvm-transforms-instcombine

# Fix the issue, update the PR
git commit --amend  # (or a new commit)
git push origin my-feature --force-with-lease
```

### Review Expectations

**Timeline**: reviews typically take 1–2 weeks. Complex changes (new passes, language features, significant refactoring) may take longer. If a review is not responded to within 2 weeks, it is acceptable to post a gentle ping comment.

**Who reviews**: anyone can review and comment; the `CODEOWNERS` bot adds specific reviewers. Comments from senior contributors carry more weight, but all technical objections must be addressed. A PR is ready to merge when:
- All CI checks pass.
- All reviewer comments are resolved (or responded to with a convincing counter-argument).
- At least one approving review from a committer.

For new contributors (without commit access), two approving reviews are typically required before a committer merges the PR.

**Review etiquette**:
- Respond to every comment, even if just "Done." or "I disagree because...".
- Avoid force-pushing after reviews start; create new commits for revisions (easier for reviewers to see what changed).
- The reviewer's job is to improve the change, not gatekeep; take feedback as guidance, not criticism.
- If a reviewer seems wrong, ask for clarification politely before concluding the review is incorrect.

### Commit Access

After several merged contributions (typically 5–10), a community member may nominate you for commit access. Commit access grants the ability to merge PRs (your own and others') directly. The process:
1. A community member posts a nomination to Discourse.
2. A brief discussion period (no objections = granted).
3. Infrastructure team adds you to the GitHub organization.

With commit access, you are expected to:
- Respond to CI failures on your commits promptly.
- Revert quickly if your change breaks something.
- Review PRs in your area of expertise.

---

## 176.5 Writing an RFC

### When to Write an RFC

An RFC (Request for Comments) is appropriate for:
- **New compiler passes**: adding a new optimization pass to `opt` or a new Clang warning.
- **New MLIR dialects**: the bar here is particularly high (§below).
- **Semantic changes**: changes to LLVM IR semantics (new instruction, changed attribute meaning, memory model changes).
- **ABI changes**: changes to calling conventions, struct layouts, or the C API.
- **New language features in Clang**: new C++ extensions, new pragmas, new attributes.
- **Large refactors**: significant restructuring of a sub-system.

When in doubt, post a short Discourse thread first ("is this worth an RFC?"). The response will quickly indicate the community's level of interest and concern.

### RFC Format

Post on Discourse in the relevant category (`Clang Frontend`, `MLIR`, `LLVM`, etc.) with `[RFC]` in the title. Recommended structure:

```markdown
# [RFC] Add a New InstCombine Fold for (X & Mask) == Mask

## Motivation

[Why does this matter? What problem does it solve? Who benefits?]

InstCombine currently misses the fold (X & Mask) == Mask -> (X & Mask) == Mask,
which is common in bit-manipulation code. This prevents further simplifications
in downstream passes.

## Proposed Design

[What are you proposing? Be specific about the implementation.]

Add a rule in InstCombineCompares.cpp:
- Match: (icmp eq (and %x, C_mask), C_mask)
- Replace: (icmp eq (and %x, C_mask), C_mask)  [simplified form]
Precondition: C_mask is a constant.
Alive2 verification: ...

## Alternatives Considered

[What other approaches were considered? Why were they rejected?]

Alternative: handle this in ValueTracking's knownBits. This would be more
general but would require changes across many passes.

## Impact on Existing Users

[Will this change affect existing programs? Any compatibility concerns?]

New optimization: may change generated code for affected patterns.
No IR breaking changes. All existing tests pass.

## Implementation Plan

[Who will implement it? What is the timeline?]

I will implement this as part of a Google Summer of Code project.
Prototype available at: [link to branch]
```

### RFC for New MLIR Dialects

Adding a new MLIR dialect faces higher scrutiny because dialects are permanent commitments — once code depends on a dialect, removal is very disruptive. The RFC must address:

1. **Need**: why is an in-tree dialect needed? Could this be an out-of-tree dialect in a downstream project?
2. **Scope**: precisely what operations will the dialect contain? What are the invariants?
3. **Lowering story**: how will this dialect be lowered to existing dialects or to LLVM IR?
4. **Maintenance**: who will maintain this dialect long-term?
5. **Design completeness**: the initial implementation should be complete enough to be useful; half-finished dialects create technical debt.

MLIR dialect RFCs often involve weeks of discussion and multiple revisions before reaching consensus.

---

## 176.6 Release Process

### Release Cycle

LLVM uses a **6-month release cycle** with major releases in April and October:
- LLVM 22.1.0: April 2026.
- LLVM 23.0.0: October 2026.
- LLVM 23.1.0: April 2027.

The `.1` releases are typically just the initial release of the 22.x branch; subsequent `.2`, `.3` etc. are bug-fix-only point releases.

### Branching and the Branch Lifecycle

Six weeks before the release date:
1. A release branch (`release/22.x`) is cut from `main`.
2. Main development continues on `main` (toward the next release).
3. Only **critical bug fixes** are cherry-picked to the release branch.

To request a cherry-pick to the release branch:
1. File a GitHub issue or tag the PR with the `release:cherry-pick` label.
2. The release manager reviews the fix and decides whether to cherry-pick.
3. Cherry-picks are conservative: only bug fixes that are isolated, well-tested, and low-risk are accepted.

### Release Management

Each release has a designated **Release Manager** from the community (announced on Discourse). The RM coordinates:
- Setting the branch date.
- Accepting cherry-pick requests.
- Running release candidate builds.
- Coordinating with distribution maintainers (Debian, Fedora, Homebrew, etc.).
- Announcing the final release.

The RM role rotates among community members; the rotation is announced on Discourse. Volunteering to be RM for a release is a significant contribution to the project.

### What "Release" Means

LLVM releases source code tarballs. The LLVM project does not maintain binary packages directly; binary packages are provided by:
- **Distributions**: Debian, Ubuntu, Fedora, Arch Linux maintain LLVM packages.
- **Homebrew/MacPorts**: macOS package managers.
- **LLVM's apt repository**: `apt.llvm.org` provides Debian/Ubuntu packages directly from the LLVM CI.
- **GitHub Releases**: source tarballs and some pre-built binaries.

The `llvm.sh` script from apt.llvm.org is the recommended way to install a specific LLVM version on Ubuntu:
```bash
wget https://apt.llvm.org/llvm.sh
chmod +x llvm.sh
sudo ./llvm.sh 22   # Install LLVM 22
```

### Deprecation Policy

LLVM uses a two-release deprecation cycle:
1. **Deprecate**: mark a feature/API as deprecated (add a deprecation note to documentation, optionally emit a warning).
2. **Remove**: remove in the next major release (6 months later).

The C API has stricter deprecation: breaking changes to the C API require broader community consensus because they affect all language bindings. The process:
1. Post RFC on Discourse.
2. Announce the deprecation in the release notes.
3. Allow one full release cycle before removal.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **GitHub Merge Queue adoption**: LLVM's CI infrastructure is migrating toward GitHub's native merge queue (replacing the phabricator-era `arc land` workflow). The October 2026 release cycle is expected to consolidate pre-merge testing into a single merge queue that serializes commits through Buildkite, reducing the frequency of broken-main incidents ([discourse.llvm.org/t/rfc-merge-queue](https://discourse.llvm.org/)).
- **LLVM 23.0 release cadence shifts**: the community has discussed moving to a rolling-branch model where point releases (`.1`, `.2`) are cut on a fixed 8-week cadence rather than on-demand. The LLVM 23.x series starting October 2026 is the candidate for piloting this approach.
- **Contributor onboarding automation**: work is underway to automate `good first issue` triage using an LLM-assisted labeling bot that scans new issues for complexity heuristics and code ownership, reducing the manual burden on `CODEOWNERS` reviewers.
- **clang-tidy LLVM-specific checks expansion**: RFC discussions target adding `llvm-use-error-handling`, `llvm-no-raw-new-delete`, and `llvm-prefer-llvm-string-ref` checks that flag common new-contributor mistakes, with enforcement via CI by LLVM 23.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Structured RFC process with formal templates**: the LLVM community is actively discussing a formalized RFC lifecycle (draft → review → accepted/rejected → implementation tracking) analogous to Rust's RFC process, with a dedicated `rfcs/` directory in the monorepo for machine-readable RFC state. A prototype governance RFC was posted to discourse.llvm.org in early 2026.
- **Sub-project governance charters**: MLIR, Clang, Flang, and libc are expected to adopt explicit governance charters (modeled on MLIR's informal practices) that document escalation paths, reviewer responsibilities, and sub-project-specific contribution requirements. This addresses recurring friction where contributors don't know which reviewers have authority.
- **Reproducible build infrastructure**: LLVM's CI is moving toward fully hermetic, reproducible builds (using Bazel and/or CMake presets with locked dependency manifests), making it easier to reproduce CI failures locally without matching the exact CI environment. Target: all pre-commit CI jobs reproducible by LLVM 25 (April 2027).
- **C API stability guarantees**: the long-standing community discussion about providing a stable, versioned C API subset (analogous to LLVM-C's existing surface but with explicit stability commitments) is expected to produce an RFC-to-implementation cycle. This would allow language binding maintainers (Rust `llvm-sys`, Julia, Swift) to build against a stable interface without rebuilding on every LLVM release.

### 5-Year Horizon (Long-Term, by ~2031)

- **Contributor attribution and credit systems**: LLVM is projected to adopt a formal contributor credit system (mapping commits to sub-projects, tracking review load, and generating per-contributor statistics) integrated with the LLVM Foundation's recognition programs. This addresses growing concerns about invisible review labor as the project scales.
- **Automated correctness infrastructure for contributions**: Alive2 (the LLVM IR peephole verifier) and KnownBitsVerifier are expected to become mandatory pre-submit checks for all InstCombine, DAGCombine, and LLVM IR semantic changes, with the CI infrastructure automatically invoking Alive2 on changed fold rules and blocking merge on verification failure.
- **Cross-project RFC federation**: as MLIR dialects are increasingly co-developed by multiple organizations (OpenXLA, Mojo/Modular, Intel MLIR extensions, SPIR-V), the contribution process is expected to evolve toward a federated RFC model where downstream projects can file binding objections during the RFC comment period, formalized through the MLIR Steering Committee structure introduced in 2025.

---

## Chapter Summary

- LLVM is governed by community consensus (not a BDFL); technical decisions happen through Discourse discussion and GitHub pull request reviews; `CODEOWNERS` identifies relevant reviewers.
- Start with GitHub Issues labeled `good first issue` or fixing `TODO`/`FIXME` comments; these are deliberately chosen for new contributors.
- Development setup: fork the repo, configure a `RelWithDebInfo` build with assertions, target only needed architectures, use `lld` for faster linking.
- Code style: `clang-format -style=LLVM` enforced by CI; no RTTI, no exceptions; use `llvm::StringRef`, `llvm::ArrayRef`, `llvm::Error` over standard equivalents.
- The review process: PRs reviewed by `CODEOWNERS` reviewers; CI runs GitHub Actions + Buildkite; all reviewer comments must be addressed; ~1–2 weeks for typical patches.
- Commit access is granted after ~5–10 merged contributions; grants merge rights and responsibility for reverting breaking changes promptly.
- RFCs are required for new passes, new dialects, semantic changes, and ABI changes; new MLIR dialects face the highest bar (must demonstrate need, complete design, and long-term maintenance commitment).
- LLVM releases every 6 months (April and October); branches are cut 6 weeks before; only critical bug fixes are cherry-picked to release branches; Release Manager role rotates among community members.


---

@copyright jreuben11
