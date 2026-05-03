# Chapter 198 — Value Tracking Infrastructure in LLVM

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

Every optimization pass faces the same fundamental question: what can we prove about this value at compile time, without executing the program? LLVM's answer is a family of interoperating libraries — collectively called **value tracking** — that propagate facts about integer bits, numeric ranges, demanded output bits, and conditional constraints through the SSA def-use graph. These libraries underpin InstCombine, InstructionSimplify, ScalarEvolution, JumpThreading, CorrelatedValuePropagation, vectorization heuristics, and dozens of backend lowering decisions. This chapter dissects the value-tracking infrastructure layer by layer: the core `ValueTracking` library, the `KnownBits` lattice, `ConstantRange` arithmetic, `DemandedBits` analysis, `LazyValueInfo`, the `SimplifyQuery` context struct, pointer alignment inference, and finally a complete worked example of a pass that consumes these APIs.

---

## 198.1 The ValueTracking Library

### 198.1.1 Role and Scope

[`llvm/include/llvm/Analysis/ValueTracking.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/ValueTracking.h) declares roughly 80 free functions that answer predicate questions about `Value*` objects without modifying IR. The implementation lives in [`llvm/lib/Analysis/ValueTracking.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/ValueTracking.cpp), a 5 000+ line file that is among the most-touched files in the entire project.

The library is intentionally **stateless and pass-agnostic**: every function takes a value, an optional context instruction, and an optional `SimplifyQuery` bundle; none of them cache results. Passes that need memoization (like InstCombine's worklist loop) call the functions on-demand per rewrite attempt. Functions that need deeper analysis build on each other recursively up to `MaxAnalysisRecursionDepth = 6` hops through the def-use graph — deep enough to catch most patterns, shallow enough to avoid exponential blowup.

The library's central constant, defined in the header, is:

```cpp
// llvm/include/llvm/Analysis/ValueTracking.h
constexpr unsigned MaxAnalysisRecursionDepth = 6;
constexpr unsigned MaxLookupSearchDepth = 10; // for getUnderlyingObject
```

When the recursion depth reaches `MaxAnalysisRecursionDepth`, the function conservatively returns the bottom of the lattice — all bits unknown for `KnownBits`, the full set for `ConstantRange`. This is always safe: approximating the result conservatively means the optimizer does fewer transformations, not incorrect ones.

Beyond bit and range queries, `ValueTracking` also provides:

- **Object identity queries**: `getUnderlyingObject(V, MaxLookupSearchDepth)` — strip GEPs and bitcasts to find the base alloca, global, or argument. Used heavily by alias analysis.
- **Dereferenceability**: `isDereferenceablePointer(V, Ty, DL, CxtI, DT, AC)` — can we load from this pointer without undefined behaviour?
- **Non-null**: `isKnownNonNull(V, DL)` — guaranteed non-null at any program point.
- **Floating-point class**: `computeKnownFPClass(V, SQ, InterestedClasses)` — which of {NaN, +Inf, -Inf, +0, -0, subnormal, normal} can the value belong to?
- **Captures and escapes**: `PointerMayBeCaptured(V, ...)` — does a pointer escape to an unknown callee?

These are consumed by alias analysis, the memory dependence graph, and loop-invariant code motion to determine whether it is safe to hoist or sink loads.

### 198.1.2 Relationship to Sibling Libraries

Value tracking answers *predicate* queries about a specific value at a specific point. Three adjacent libraries share a complementary relationship:

| Library | What it does |
|---------|-------------|
| `ValueTracking` | Answers boolean or lattice-valued questions about a `Value*` |
| `InstructionSimplify` | Returns a simplified `Value*` or null (algebraic identities, constant folding) |
| `ConstantFolding` | Folds all-constant operands to a `Constant*` |
| `InstCombine` | Rewrites instructions using `ValueTracking` + `InstructionSimplify` results |

`InstructionSimplify` calls `ValueTracking` to justify simplifications; `InstCombine` calls both. The flow is one-directional: `ValueTracking` never calls the simplification layer (that would risk circular reasoning).

The dependency hierarchy is:

```
ConstantFolding      (only handles constant inputs)
       |
InstructionSimplify  (algebraic identities, uses ValueTracking for non-const facts)
       |
ValueTracking        (bit/range/predicate queries — no IR modification)
       |
KnownBits / ConstantRange / DemandedBits / LazyValueInfo
       |
AssumptionCache / DominatorTree / SimplifyQuery context
```

`InstCombine` sits above all of them: it calls `InstructionSimplify` to attempt a complete simplification, `ValueTracking` to gather facts that justify structural rewrites, and finally `ConstantFolding` when all operands are known constants after simplification.

For the new pass manager context, see [Ch59 — The New Pass Manager](../part-10-analysis-middle-end/ch59-the-new-pass-manager.md) and [Ch62 — Scalar Optimizations](../part-10-analysis-middle-end/ch62-scalar-optimizations.md).

---

## 198.2 KnownBits

### 198.2.1 The Struct

[`llvm/include/llvm/Support/KnownBits.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/KnownBits.h) defines:

```cpp
struct KnownBits {
  APInt Zero;  // bits known to be 0
  APInt One;   // bits known to be 1
  // invariant: Zero & One == 0 (no bit can be simultaneously known 0 and 1)
};
```

Both `APInt` fields are the same width as the value they describe. A bit position `i` is:

- **known zero**: `Zero[i] == 1`
- **known one**: `One[i] == 1`
- **unknown**: `Zero[i] == 0 && One[i] == 0`

The invariant `Zero & One == 0` is enforced by construction. A `KnownBits` that violates it (via `hasConflict()`) signals that the value is poison. Important predicates:

```cpp
bool isConstant() const; // all bits known — Zero.popcount()+One.popcount()==width
bool isZero()     const; // Zero.isAllOnes()
bool isAllOnes()  const; // One.isAllOnes()
bool isUnknown()  const; // Zero.isZero() && One.isZero()
```

### 198.2.2 The Lattice Structure

`KnownBits` is a product lattice — one two-element lattice per bit position — ordered by information content. For bit `i`:

```
          ⊤ (known, hasConflict: Zero[i]=1 AND One[i]=1 → poison)
         / \
     zero   one
       \   /
        ⊥ (unknown: Zero[i]=0 AND One[i]=0)
```

The join (`⊔`) operation is `intersectWith`: the result contains only bits that are known in *both* inputs. This is the correct operation for `phi` nodes — the output is only known if it is known on every incoming edge. The meet (`⊓`) operation is `unionWith`: the result contains all bits known in *either* input, used when merging constraints from different information sources (e.g., `!range` metadata AND `@llvm.assume`).

The `fromKnownBits` / `toKnownBits` conversions between `KnownBits` and `ConstantRange` form a Galois connection: `ConstantRange::fromKnownBits(KB, IsSigned)` returns the smallest `ConstantRange` consistent with `KB`, and `KnownBits::fromConstantRange(CR, IsSigned)` returns the most precise `KnownBits` derivable from a range.

### 198.2.3 computeKnownBits

The central function is:

```cpp
// llvm/include/llvm/Analysis/ValueTracking.h
KnownBits computeKnownBits(const Value *V, const SimplifyQuery &Q,
                            unsigned Depth = 0);
```

(An older overload taking `DataLayout`, `AssumptionCache`, `Instruction*`, and `DominatorTree` separately still exists for call sites that don't have a `SimplifyQuery` assembled; prefer the `SimplifyQuery` form in new code.)

The function walks the def-use graph recursively:

- **Constants**: trivially fills `One` from the constant bits, `Zero` from their complement.
- **Arguments and loads**: only alignment metadata and `!range` metadata contribute; the rest is unknown.
- **`and`**: `Known.Zero |= KnownRHS.Zero | KnownLHS.Zero; Known.One &= KnownLHS.One & KnownRHS.One;`
- **`or`**: `Known.One |= ...; Known.Zero &= ...;`
- **`xor`**: known-one bits arise where exactly one operand is known-one; known-zero where both operands have the same known value.
- **`shl k`**: shifts `One` and `Zero` left by `k`, inserting zeros from the right.
- **`lshr k`**: shifts right, inserting zeros from the left.
- **`ashr k`**: replicate the known sign bit into the vacated positions.
- **`trunc`**: simply keeps the low bits.
- **`zext`**: widens with known-zero high bits.
- **`sext`**: if the sign bit is known, replicates it into extended bits.
- **`add`**: uses carry propagation — if there are trailing known-zero bits on both operands, those many trailing result bits are known zero.
- **`mul`**: trailing known-zero count is the sum of trailing known-zero counts of both operands.
- **`phi`**: intersects (`&`) the known bits across all incoming values (only bits known-zero in *every* predecessor are known-zero overall).

### 198.2.4 Concrete Example

Consider the expression `(x & 0xFF) | 0x100` for an `i32` value `x`.

```llvm
; Example IR (i32 arithmetic)
%masked = and i32 %x, 255        ; 0x000000FF
%result = or  i32 %masked, 256   ; 0x00000100
```

`computeKnownBits` for `%result` proceeds as follows:

**Step 1 — compute KnownBits for `%masked = and i32 %x, 255`:**

- `%x` is an argument; all bits unknown: `Zero_x = 0x00000000`, `One_x = 0x00000000`.
- Constant `255 = 0x000000FF`: `Zero_c = 0xFFFFFF00`, `One_c = 0x000000FF`.
- `and` rule: `Known.Zero = Zero_x | Zero_c = 0xFFFFFF00`; `Known.One = One_x & One_c = 0x00000000`.
- Result: bits 8–31 known zero, bits 0–7 unknown.

**Step 2 — compute KnownBits for `%result = or i32 %masked, 256`:**

- `%masked`: `Zero_m = 0xFFFFFF00`, `One_m = 0x00000000`.
- Constant `256 = 0x00000100`: `Zero_256 = 0xFFFFFEFF`, `One_256 = 0x00000100`.
- `or` rule: `Known.One = One_m | One_256 = 0x00000100`; `Known.Zero = Zero_m & Zero_256 = 0xFFFFFE00`.
- Bit 8 is now known-one (set by the constant 256). Bits 9–31 remain known-zero (both operands have them known-zero). Bits 0–7 are unknown.

**Final result**: Known.Zero = `0xFFFFFE00`, Known.One = `0x00000100`. The value lives in `[256, 511]`.

This result enables several downstream optimizations:
- A comparison `%result >= 256` folds to `true` (bit 8 is always set).
- A comparison `%result > 511` folds to `false` (bits 9–31 are always zero).
- A truncation to `i16` is lossless (bits 16–31 known zero).
- The `zext` of `trunc i16 %result` to `i32` folds back to `%result`.

### 198.2.5 ComputeNumSignBits and Scalar Predicates

```cpp
// Returns the number of bits equal to the sign bit (always >= 1)
unsigned ComputeNumSignBits(const Value *Op, const DataLayout &DL,
                             AssumptionCache *AC = nullptr,
                             const Instruction *CxtI = nullptr,
                             const DominatorTree *DT = nullptr,
                             bool UseInstrInfo = true,
                             unsigned Depth = 0);
```

`ComputeNumSignBits` returns the count of high-order bits that are all equal to the sign bit (including the sign bit itself — hence the result is always at least 1). For an `ashr i32 %x, 2`, the result has the sign bit replicated into bits 31, 30, and 29, so `ComputeNumSignBits` returns 3.

This function drives several important optimizations:

- **Backend narrowing**: AArch64's `SMULL`/`UMULL` instructions require that inputs fit in 32 signed bits of a 64-bit register. `ComputeNumSignBits` proves that a 64-bit value sign-extended from a 32-bit operation satisfies this without an explicit `sext`.
- **InstCombine `sext`/`ashr` folding**: `sext (ashr i8 %x, 4) to i32` rewrites to `ashr (sext i8 %x to i32), 4` because the outer `sext` is redundant — the `ashr` already replicates the sign bit enough times.
- **Division strength reduction**: if `ComputeNumSignBits(x) > 1`, then `x` cannot be `INT_MIN`, which means `sdiv x, -1` cannot overflow and may be strength-reduced.

High-level boolean predicates built on `computeKnownBits`:

```cpp
bool isKnownNonZero(const Value *V, const SimplifyQuery &Q, unsigned Depth=0);
bool isKnownNonNegative(const Value *V, const SimplifyQuery &SQ, unsigned Depth=0);
bool isKnownPositive(const Value *V, const SimplifyQuery &SQ, unsigned Depth=0);
bool isKnownNegative(const Value *V, const SimplifyQuery &SQ, unsigned Depth=0);
bool MaskedValueIsZero(const Value *V, const APInt &Mask,
                       const SimplifyQuery &SQ, unsigned Depth=0);
```

`isKnownNonZero` is notably more powerful than a simple `One.isNonZero()` check: it also handles cases where a value is non-zero by structural argument (a `or` with a non-zero constant, a `shl nsw` of a non-zero value, or a value proved non-zero by `@llvm.assume`). The implementation in `ValueTracking.cpp` has a dedicated path for these structural cases before falling back to `computeKnownBits`.

`isKnownNonNegative` checks `Known.Zero.isSignBitSet()` — the sign bit is in the `Zero` field — along with `nsw` flags and assume constraints.

### 198.2.6 Use in InstCombine

`InstCombine`'s `simplifyAndInst` (in [`llvm/lib/Transforms/InstCombine/InstCombineAndOrXor.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/InstCombine/InstCombineAndOrXor.cpp)) calls `MaskedValueIsZero` to detect when an `and` mask is redundant:

```cpp
// Simplify: (and V, Mask) -> V  when all Mask bits are already zero in V
if (MaskedValueIsZero(Op0, *C, SQ))
  return replaceInstUsesWith(I, Op0);
```

More concretely, `(x & 0xFF) & 0xFF` folds to `x & 0xFF` because `computeKnownBits` shows bits 8–31 of `x & 0xFF` are already zero, making the outer `& 0xFF` redundant for bits 0–7 and a no-op for the others.

A richer catalogue of `KnownBits`-driven InstCombine patterns:

| Pattern | Condition | Result |
|---------|-----------|--------|
| `(x << k) >> k` | `lshr` | `x & ((1<<(width-k))-1)` if sign bit unknown; `x` if top k bits known zero |
| `x | C` | all bits of C already known-one in x | `x` |
| `x ^ C` | all bits of C already known-zero in x | `x` |
| `icmp eq (x & mask), 0` | all mask bits already known-zero in x | `true` |
| `sext (trunc x to i8) to i32` | top 24 bits already equal sign of bit 7 | `x` |
| `add nsw x, y` | neither operand can carry into the sign bit | retain `nsw`, enable range narrowing |

Each of these is a single call to `computeKnownBits` followed by a bitwise test. The key insight is that these rewrites are *safe by the lattice*: the `KnownBits` result is a conservative over-approximation (it may claim bits are unknown when they are actually constant), so any rewrite justified by the result is sound for all possible runtime values.

---

## 198.3 ConstantRange

### 198.3.1 Representation

[`llvm/include/llvm/IR/ConstantRange.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/ConstantRange.h) represents the set of values a fixed-width integer can take as a **half-open interval `[Lower, Upper)`** over the wrap-around number line. The wrap-around semantics mean:

- If `Lower < Upper` (no wrap): the set is `{Lower, Lower+1, ..., Upper-1}`.
- If `Lower > Upper` (wrapping): the set is `{Lower, ..., MaxVal, 0, ..., Upper-1}`.
- **Full set**: `Lower == Upper` (with both equal to any value, conventionally 0).
- **Empty set**: represented as `(BitWidth, false)` — `Lower == Upper == 0` by a dedicated sentinel.
- **Single element**: `Upper == Lower + 1`.

```cpp
// Constructors
ConstantRange(uint32_t BitWidth, bool isFullSet); // full or empty
ConstantRange(APInt Value);                        // single-element
ConstantRange(APInt Lower, APInt Upper);           // explicit range

// Factory helpers
static ConstantRange getFull(uint32_t BitWidth);
static ConstantRange getEmpty(uint32_t BitWidth);
static ConstantRange fromKnownBits(const KnownBits &Known, bool IsSigned);
```

### 198.3.2 Set Operations

`ConstantRange` supports the standard set operations needed for dataflow analysis:

```cpp
// Union: smallest range containing all elements of either (may lose precision)
ConstantRange unionWith(const ConstantRange &CR,
                        PreferredRangeType Type = Smallest) const;

// Intersection: largest range contained in both
ConstantRange intersectWith(const ConstantRange &CR,
                             PreferredRangeType Type = Smallest) const;

// Signed/unsigned containment
bool contains(const APInt &Val) const;
bool contains(const ConstantRange &CR) const;

// Check predicate between two ranges (does every x in this satisfy "x op y" for some y in Other?)
bool icmp(CmpInst::Predicate Pred, const ConstantRange &Other) const;
```

`unionWith` is the join operator in the lattice of ranges ordered by set containment. `intersectWith` is the meet. For `phi` nodes, the analysis joins (unions) the ranges from all predecessors. For branch conditions, it intersects the value's range with the range implied by the predicate.

The `PreferredRangeType` enum controls which of the two complementary representations is preferred when the exact answer is not representable as a single `[Lower, Upper)` interval. `Smallest` picks the contiguous range containing the fewest elements; `Unsigned` and `Signed` prefer the non-wrapping half of the number line.

### 198.3.3 Arithmetic Operations

`ConstantRange` supports all integer operations with wrapping semantics:

```cpp
ConstantRange add(const ConstantRange &Other) const;
ConstantRange sub(const ConstantRange &Other) const;
ConstantRange multiply(const ConstantRange &Other) const;
ConstantRange shl(const ConstantRange &Other) const;
ConstantRange lshr(const ConstantRange &Other) const;
ConstantRange ashr(const ConstantRange &Other) const;
ConstantRange udiv(const ConstantRange &Other) const;
ConstantRange sdiv(const ConstantRange &Other) const;
ConstantRange urem(const ConstantRange &Other) const;
ConstantRange srem(const ConstantRange &Other) const;
```

Each function returns the tightest representable `ConstantRange` containing all possible results. For example:

```cpp
// [0, 256) shl [2, 3) = ?
// max result: 255 << 2 = 1020, but we need the full [0, 1024) to be safe
ConstantRange LHS(APInt(32, 0), APInt(32, 256));    // [0, 256)
ConstantRange ShiftAmt(APInt(32, 2), APInt(32, 3)); // exactly 2 (shift by 2)
ConstantRange Result = LHS.shl(ShiftAmt);            // [0, 1024)
```

The shift multiplies the upper bound: `256 * 4 = 1024`, so the result is `[0, 1024)`. The result is exact in this case because the shift amount is a single value.

### 198.3.4 No-Wrap Variants and makeGuaranteedNoWrapRegion

The no-wrap variants tighten the result using NSW/NUW flags:

```cpp
ConstantRange addWithNoWrap(const ConstantRange &Other, unsigned NoWrapKind,
                             PreferredRangeType RangeType = Smallest) const;
ConstantRange subWithNoWrap(const ConstantRange &Other, unsigned NoWrapKind,
                             PreferredRangeType RangeType = Smallest) const;
ConstantRange multiplyWithNoWrap(const ConstantRange &Other, unsigned NoWrapKind,
                                  PreferredRangeType RangeType = Smallest) const;
```

`NoWrapKind` is a bitmask of `OverflowingBinaryOperator::NoUnsignedWrap` and `NoSignedWrap`.

`makeGuaranteedNoWrapRegion` works in reverse: given a binary operation and one known operand range, what range of the *other* operand is guaranteed not to overflow?

```cpp
static ConstantRange makeGuaranteedNoWrapRegion(
    Instruction::BinaryOps BinOp,
    const ConstantRange &Other,
    unsigned NoWrapKind);
```

This is used by ScalarEvolution to prove that an induction variable step never overflows.

### 198.3.5 Use in ScalarEvolution and InstructionSimplify

`ScalarEvolution` (see [Ch61 — Foundational Analyses](../part-10-analysis-middle-end/ch61-foundational-analyses.md)) maintains a `ConstantRange` for every SCEV node. For an `AddRecExpr {start, +, step}` with a known iteration count `N`, it computes the range as:

```
[start, start + step * N)   (unsigned, when step > 0 and no overflow)
```

`InstructionSimplify` uses `ConstantRange::icmp` to fold comparisons: if `LHS_range.icmp(ICmpInst::ICMP_ULT, RHS_range)` returns `true` — meaning every value in LHS is less than every value in RHS — the compare folds to `true`, enabling branch elimination.

A concrete ScalarEvolution scenario:

```llvm
; Loop with i32 induction variable i, 0 to 99
; SE computes SCEV for i as AddRecExpr {0, +, 1}<nuw><nsw>
; ConstantRange for the AddRec over 100 iterations: [0, 100)
for.body:
  %i = phi i32 [ 0, %entry ], [ %i.next, %for.body ]
  ; SE knows %i is in [0, 100)
  %cmp = icmp ult i32 %i, 200   ; always true: [0,100) ult [200,201) → fold to true
  br i1 %cmp, label %safe, label %trap  ; %trap is dead
  %i.next = add nuw nsw i32 %i, 1
  %done = icmp eq i32 %i.next, 100
  br i1 %done, label %exit, label %for.body
```

The `ConstantRange` for `%i` is `[0, 100)`. `icmp ult [0,100) [200,201)` is provably true, so `InstructionSimplify` replaces the branch condition with `true` and the `%trap` block becomes dead code eligible for elimination.

`ConstantRange` is also used to validate no-wrap flags. `makeGuaranteedNoWrapRegion(Instruction::Add, {1}, NUW)` returns `[0, UINT_MAX)` — the set of LHS values for which adding 1 cannot unsigned-overflow. ScalarEvolution calls this when attempting to prove that an induction variable increment with `nuw` is valid throughout the loop.

Another use: `!range` metadata on load instructions. When LLVM lowers a C++ `unsigned char` load, it can attach `!range !{ i32 0, i32 256 }`. `computeKnownBits` and `ConstantRange` both honor this metadata, allowing downstream passes to treat the loaded value as if it were masked to `[0, 256)` without examining the actual store sites.

---

## 198.4 DemandedBits

### 198.4.1 The Analysis

[`llvm/include/llvm/Analysis/DemandedBits.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/DemandedBits.h) implements a backward dataflow analysis that computes, for each instruction, which output bits are *demanded* — consumed by at least one user.

```cpp
class DemandedBits {
public:
  DemandedBits(Function &F, AssumptionCache &AC, DominatorTree &DT);

  // Which bits of instruction I's output are demanded by its users?
  APInt getDemandedBits(Instruction *I);

  // Which bits of the value flowing through use U are demanded?
  APInt getDemandedBits(Use *U);

  // Has the instruction been proved dead (zero demanded bits)?
  bool isInstructionDead(Instruction *I);
  bool isUseDead(Use *U);
};
```

The analysis starts at "root" uses (stores, branches, calls, returns — anything that ultimately consumes a value) and propagates backward through the def-use graph, tracking which bits of each instruction's output are needed. The transfer functions are the bitwise inverses of `computeKnownBits`'s forward rules: for an `and` instruction, the demanded bits of the LHS operand are `demanded_output & ~known_one_of_RHS`.

The propagation rules for the most common instructions:

| Instruction | Demanded bits of output `AOut` | Demanded bits of operand |
|-------------|-------------------------------|--------------------------|
| `trunc i32 → i16` | bits 0–15 | AOut (extended to 32) |
| `zext i16 → i32` | high 16 demanded? | only low 16 of AOut |
| `and i32 %a, C` | AOut | AOut & C (for %a) |
| `or i32 %a, C` | AOut | AOut & ~C (for %a) |
| `shl i32 %a, k` | AOut | AOut >> k (logical shift) |
| `lshr i32 %a, k` | AOut | AOut << k |
| `add` | AOut | computed via `determineLiveOperandBitsAdd` |

The analysis runs to a fixed point. On entry to the computation, the demanded-bits of the operands of stores, branches, and calls are the full bit-width (all bits demanded). The backward propagation then narrows demand through `trunc`, mask, and shift operations.

In the new pass manager:

```cpp
auto &DB = AM.getResult<DemandedBitsAnalysis>(F);
APInt demanded = DB.getDemandedBits(&I);
```

### 198.4.2 Use in InstCombine and CodeGenPrepare

Consider:

```llvm
%t1 = or  i32 %x, 65280          ; 0xFF00
%t2 = and i32 %t1, 255            ; 0x00FF
```

`getDemandedBits` for `%t1` (as used by `%t2`) returns `APInt(32, 0xFF)` — only bits 0–7 are demanded. The `or` with `0xFF00` sets bits 8–15, none of which are demanded. `InstCombine` can therefore simplify the sequence to `%t2 = and i32 %x, 255`, eliminating the `or` entirely.

`CodeGenPrepare` uses `DemandedBits` to decide whether to widen narrow operations: if only 8 bits of a 32-bit result are ever demanded, there is no point materializing the full 32-bit operation on a platform where 8-bit ops are cheaper.

The helper statics `DemandedBits::determineLiveOperandBitsAdd` and `determineLiveOperandBitsSub` compute which bits of an addition's operands are necessary given a demanded-output mask and known bits of the other operand — the key transfer function for propagating demands through carry chains.

`DemandedBits` differs from the simpler `SimplifyDemandedBits` path inside InstCombine. `SimplifyDemandedBits` is a recursive procedure called *during* an InstCombine rewrite attempt: it takes a demanded-bits mask as an argument and recursively simplifies operands, potentially creating new instructions. `DemandedBits` as an analysis pass pre-computes demanded bits for the *entire function* in a single pass, then stores results in a `DenseMap<Instruction*, APInt>`. The pre-computed analysis is cheaper when many instructions are queried (as in CodeGenPrepare), while the recursive `SimplifyDemandedBits` is preferred for InstCombine's single-instruction context where the analysis would otherwise be called once per candidate instruction.

The two mechanisms cooperate: `SimplifyDemandedBits` can use the stored `DemandedBits` results to initialize its demanded mask for the starting instruction, avoiding redundant recursion when the analysis is available.

---

## 198.5 LazyValueInfo

### 198.5.1 Demand-Driven Conditional Inference

[`llvm/include/llvm/Analysis/LazyValueInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/LazyValueInfo.h) provides **context-sensitive, demand-driven** value inference. Unlike `computeKnownBits` (which is flow-insensitive), `LazyValueInfo` exploits branch conditions and edge guards to narrow value ranges at specific program points.

```cpp
class LazyValueInfo {
public:
  // Is V a known constant at instruction CxtI?
  Constant *getConstant(Value *V, Instruction *CxtI);

  // What ConstantRange holds for V at instruction CxtI?
  ConstantRange getConstantRange(Value *V, Instruction *CxtI,
                                 bool UndefAllowed);

  // Is the predicate "V op C" provably true/false at CxtI?
  Constant *getPredicateAt(CmpInst::Predicate Pred, Value *V, Constant *C,
                            Instruction *CxtI, bool UseBlockValue);

  // Is the predicate "LHS op RHS" provably true/false at CxtI?
  Constant *getPredicateAt(CmpInst::Predicate Pred, Value *LHS, Value *RHS,
                            Instruction *CxtI, bool UseBlockValue);

  // What does V evaluate to on the edge FromBB -> ToBB?
  Constant *getPredicateOnEdge(CmpInst::Predicate Pred, Value *V, Constant *C,
                                BasicBlock *FromBB, BasicBlock *ToBB,
                                Instruction *CxtI = nullptr);
};
```

The implementation builds a sparse lattice over `ConstantRange` values. It recurses backward through the CFG only as far as needed to answer the specific query (hence "lazy"). It merges information from:

- Branch conditions guarding the path to `CxtI`
- `@llvm.assume` intrinsics
- `!range` metadata on loads
- Predecessor block values

### 198.5.2 Use in JumpThreading

`JumpThreading` queries `LazyValueInfo` on edges. If `getPredicateAt(ICMP_EQ, %flag, ConstantInt::getTrue(), terminator, true)` returns `ConstantInt::getTrue()`, the branch condition is always taken on that edge and the pass threads the jump, duplicating the branch target.

A concrete JumpThreading scenario:

```llvm
entry:
  %cmp = icmp eq i32 %x, 42
  br i1 %cmp, label %is_42, label %not_42

is_42:
  ; %x is known to be 42 here — LVI returns ConstantRange{42, 43)
  %y = add i32 %x, 1        ; folds to 43
  %cmp2 = icmp eq i32 %y, 43 ; folds to true → branch always taken
  br i1 %cmp2, label %done, label %unreachable
```

`LazyValueInfo::getConstantRange` on `%x` at the `is_42` block returns the singleton range `[42, 43)`. Adding 1 gives `[43, 44)`. The comparison with 43 is then provably true. `JumpThreading` removes the `%unreachable` successor.

### 198.5.3 Use in CorrelatedValuePropagation

`CorrelatedValuePropagation` uses `LazyValueInfo` to narrow operations that depend on preceding comparisons. The classic example is `sdiv` or `srem`: if a preceding guard `%x >= 0` is provably true at the division site, `LVI::getConstantRange` returns a non-negative range for `%x`, allowing the backend to use faster unsigned division. The pass also:

- Removes `overflow` checks on `@llvm.sadd.with.overflow` when the range proves no overflow can occur.
- Demotes `sdiv`/`srem` to `udiv`/`urem` when both operands are proved non-negative.
- Folds `select` instructions where the condition is proved constant.
- Removes `@llvm.assume` intrinsics that are provably redundant (the fact they assert is already known to hold).

The key API pattern in CorrelatedValuePropagation is:

```cpp
// Is %x known to be in [0, INT_MAX] at this sdiv?
ConstantRange Range = LVI.getConstantRange(&X, DivInst, /*UndefAllowed=*/false);
if (Range.isAllNonNegative()) {
  // Replace sdiv with udiv — same semantics when both operands >= 0
  auto *UDiv = BinaryOperator::CreateUDiv(DivInst->getOperand(0),
                                          DivInst->getOperand(1), "", DivInst);
  DivInst->replaceAllUsesWith(UDiv);
  DivInst->eraseFromParent();
}
```

---

## 198.6 SimplifyQuery

### 198.6.1 The Context Struct

All value-tracking functions that benefit from control-flow or assumption information accept a `SimplifyQuery` (declared in [`llvm/include/llvm/Analysis/SimplifyQuery.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/SimplifyQuery.h)):

```cpp
struct SimplifyQuery {
  const DataLayout &DL;           // required — provides pointer sizes, alignment
  const TargetLibraryInfo *TLI = nullptr;  // libcall identification
  const DominatorTree *DT = nullptr;       // dominator structure for assume queries
  AssumptionCache *AC = nullptr;           // @llvm.assume intrinsics
  const Instruction *CxtI = nullptr;       // "we are analyzing at this point"
  const DomConditionCache *DC = nullptr;   // cached dominating conditions
  const CondContext *CC = nullptr;         // condition being assumed true/false
  const InstrInfoQuery IIQ;               // whether to trust nsw/nuw flags
  bool CanUseUndef = true;
};
```

Constructors:

```cpp
// Minimal — only DataLayout
SimplifyQuery SQ(DL);

// Full — with DominatorTree and AssumptionCache
SimplifyQuery SQ(DL, TLI, DT, AC, /*CxtI=*/&I);
```

### 198.6.2 How @llvm.assume Interacts with KnownBits

`@llvm.assume(cond)` tells the compiler that `cond` is non-zero at the point of the intrinsic. When `computeKnownBits` is called with an `AssumptionCache` in the `SimplifyQuery`, it iterates over all `@llvm.assume` calls that dominate `CxtI` and extracts constraints from their operands.

```llvm
define i32 @example(i32 %x) {
  %cmp = icmp sge i32 %x, 0
  call void @llvm.assume(i1 %cmp)
  ; At this point, computeKnownBits(%x, SQ) can set Zero[31] = 1
  ; because the assume guarantees the sign bit is 0.
  %result = add nsw i32 %x, 1
  ret i32 %result
}
```

The assume establishes `x >= 0`, so `computeKnownBits` sets `Known.Zero` bit 31 (the sign bit). `isKnownNonNegative` returns `true`. InstCombine can then prove that `add nsw i32 %x, 1` does not sign-overflow when `%x < INT_MAX` — a fact it can combine with the non-negativity to narrow the result range further.

More powerful constraints come through assume **operand bundles**, standardized in LLVM 14+:

```llvm
; Attach multiple constraints to a single assume
call void @llvm.assume(i1 true) [ "align"(i32* %p, i32 16),
                                   "nonnull"(i32* %p),
                                   "dereferenceable"(i32* %p, i32 64) ]
```

The `AssumeBundleQueries` utilities (in [`llvm/include/llvm/Analysis/AssumeBundleQueries.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/AssumeBundleQueries.h)) enumerate the operand bundles attached to a specific assume, filter by attribute kind, and extract the associated values. When `computeKnownBits` is called on `%p`, the `align(16)` bundle sets four trailing known-zero bits in the pointer's `KnownBits.Zero` field.

The `AssumptionCache` analysis pre-indexes all `@llvm.assume` calls in a function by the values they mention, making the lookup O(number of assumes mentioning V) rather than a full function scan. This is critical for large functions with many assumes (e.g., functions produced by `-fno-delete-null-pointer-checks` or the Clang `-fbounds-safety` feature).

**Context sensitivity**: `computeKnownBits` only uses an assume if it *dominates* the context instruction `CxtI`. An assume in a different branch of a conditional cannot be used at a point before the merge. This domination check is performed by `isValidAssumeForContext` in `ValueTracking.cpp`, which calls `DT.dominates(AssumeI, CxtI)`.

```cpp
// Typical setup in a pass:
auto &AC = AM.getResult<AssumptionAnalysis>(F);
auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
const DataLayout &DL = F.getDataLayout();

for (Instruction &I : instructions(F)) {
  SimplifyQuery SQ(DL, /*TLI=*/nullptr, &DT, &AC, &I);
  KnownBits Known = computeKnownBits(I.getOperand(0), SQ);
  // Known now reflects any @llvm.assume constraints visible at &I
}
```

---

## 198.7 Pointer Tracking and Alignment Inference

Alignment information is crucial for load/store folding and auto-vectorization decisions. LLVM tracks pointer alignment through two complementary mechanisms that interact with the value-tracking infrastructure.

### 198.7.1 Value::getPointerAlignment

**Value member function** (always available, uses DataLayout and GEP analysis):

```cpp
// llvm/include/llvm/IR/Value.h
Align Value::getPointerAlignment(const DataLayout &DL) const;
```

This returns the minimum guaranteed alignment of the pointer by inspecting:

- `alloca` instructions: the explicitly declared alignment attribute.
- Global variables: the `align` attribute in the IR (`@g = global i32 0, align 8`).
- `getelementptr` instructions: GEP offsets can increase or decrease known alignment. A GEP that adds a constant multiple of 16 to a 16-byte-aligned base is still 16-byte aligned.
- `inttoptr` and `ptrtoint`: treated conservatively unless the integer constant is known.
- Return values of `@llvm.assume(align ...)` operand bundles.

The function is entirely DataLayout-driven and does not require an `AssumptionCache` or `DominatorTree` — it is safe to call during any analysis.

### 198.7.2 getKnownAlignment

**Local.h utility** (value-tracking-driven, incorporates `@llvm.assume` and dominator info):

```cpp
// llvm/include/llvm/Transforms/Utils/Local.h
inline Align getKnownAlignment(Value *V, const DataLayout &DL,
                                const Instruction *CxtI = nullptr,
                                AssumptionCache *AC = nullptr,
                                const DominatorTree *DT = nullptr);
```

`getKnownAlignment` delegates to `getOrEnforceKnownAlignment` and additionally queries `computeKnownBits` to exploit trailing known-zero bits in the pointer value: a pointer with `N` trailing known-zero bits is at least `2^N`-byte aligned.

A concrete example: if `%ptr = and i64 %base, -16` (mask to 16-byte boundary), then `computeKnownBits(%ptr)` shows `Known.Zero = 0x...000F` (bits 0–3 known zero). `getKnownAlignment` derives alignment `= 2^4 = 16` without any explicit alignment annotation.

This is valuable for cases produced by hand-written assembly interop or intrinsic patterns where alignment is implicit in the bit manipulation rather than explicit in an `align` attribute.

### 198.7.3 Use in Vectorization and Load/Store Optimization

The loop vectorizer (`LoopVectorize.cpp`) queries `getKnownAlignment` on the base pointer of each memory access:

```cpp
// Pseudocode from LoopVectorize.cpp logic
Align PtrAlign = getKnownAlignment(BasePtr, DL, CxtI, AC, DT);
if (PtrAlign >= VectorWidth) {
  // Emit aligned vector load: load <4 x float>, ptr %base, align 16
  LI->setAlignment(Align(VectorWidth));
} else {
  // Insert runtime alignment check or use unaligned load
}
```

On x86-64, `movaps` (aligned) requires 16-byte alignment and faults on misalignment, while `movups` (unaligned) is safe but ~10% slower on modern CPUs. On SVE (AArch64) and RISC-V V, aligned loads can bypass the alignment fixup path in the hardware. The difference between knowing alignment and not knowing it is the difference between emitting a single clean vector load and emitting a scalar prologue loop to align the pointer.

The `InstCombine` pass also uses alignment information to fold `memcpy`/`memmove` calls into vector loads and stores when the source and destination are provably aligned to the natural alignment of the copy width.

---

## 198.8 Writing a Pass That Uses Value Tracking

### 198.8.1 The Goal

We want a pass that finds `add nsw` instructions where `KnownBits` proves the signed addition cannot wrap — specifically, where both operands are known non-negative and their sum is guaranteed to fit in the signed range — and **removes** the `nsw` flag. Removing `nsw` is always safe (it only widens the set of defined behaviors): the `nsw` flag asserts that signed overflow is undefined behavior, so removing it merely stops asserting that guarantee while keeping the instruction's defined semantics identical for all non-overflowing inputs. This example is deliberately chosen to avoid the harder soundness obligation of adding `nsw`. The pass demonstrates the full value-tracking API without requiring an Alive2 proof of the transformation itself.

The key insight in the implementation: if both operands of an `add i32` have their top two bits (bits 31 and 30) known zero, then each operand is in `[0, 2^30 - 1]`, and their sum is in `[0, 2^31 - 2]`, which fits in the non-negative half of `i32` without wrapping. This is a conservative but always-correct check.

### 198.8.2 Complete Pass Source

```cpp
// ValueTrackingDemo.cpp
//
// An opt plugin pass that removes redundant `nsw` flags from `add` instructions
// where KnownBits proves the operands cannot produce a signed overflow.
// Removing nsw is unconditionally sound (it only makes the result more defined).
//
// Build:
//   clang++ -std=c++20 -fPIC -shared \
//     $(llvm-config-22 --cxxflags) \
//     -I$(llvm-config-22 --includedir) \
//     ValueTrackingDemo.cpp -o ValueTrackingDemo.so
//
// Run:
//   opt-22 -load-pass-plugin=./ValueTrackingDemo.so \
//           -passes=remove-redundant-nsw \
//           input.ll -S -o output.ll

#include "llvm/Analysis/AssumptionCache.h"
#include "llvm/Analysis/SimplifyQuery.h"
#include "llvm/Analysis/ValueTracking.h"
#include "llvm/IR/Dominators.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/PassManager.h"
#include "llvm/Plugins/PassPlugin.h"
#include "llvm/Support/KnownBits.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {

struct RemoveRedundantNSW : public PassInfoMixin<RemoveRedundantNSW> {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM) {
    auto &DL = F.getDataLayout();
    auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
    auto &AC = AM.getResult<AssumptionAnalysis>(F);

    bool Changed = false;

    for (auto &BB : F) {
      for (auto &I : BB) {
        // Only examine `add nsw` binary operators.
        auto *BO = dyn_cast<BinaryOperator>(&I);
        if (!BO || BO->getOpcode() != Instruction::Add)
          continue;
        if (!BO->hasNoSignedWrap())
          continue;

        Value *LHS = BO->getOperand(0);
        Value *RHS = BO->getOperand(1);

        // Build a SimplifyQuery anchored at this instruction.
        // This allows computeKnownBits to exploit @llvm.assume calls
        // that dominate the current instruction.
        SimplifyQuery SQ(DL, /*TLI=*/nullptr, &DT, &AC, /*CxtI=*/BO);

        // Query known bits for both operands.
        KnownBits KnownLHS = computeKnownBits(LHS, SQ);
        KnownBits KnownRHS = computeKnownBits(RHS, SQ);

        // If both operands are known non-negative (sign bit = known zero)
        // and the result width can accommodate their sum without signed
        // overflow, the nsw flag is redundant.
        //
        // A conservative but correct check: if both operands have their
        // sign bit known-zero AND at least the MSB-1 bit known-zero,
        // no signed overflow is possible regardless of the lower bits.
        unsigned BitWidth = KnownLHS.getBitWidth();
        bool LHSTopTwoClear = KnownLHS.Zero.isSignBitSet() &&
                              KnownLHS.Zero.lshr(1).isSignBitSet();
        bool RHSTopTwoClear = KnownRHS.Zero.isSignBitSet() &&
                              KnownRHS.Zero.lshr(1).isSignBitSet();

        if (LHSTopTwoClear && RHSTopTwoClear) {
          // Both operands fit in the non-negative half of the type with room
          // to spare — their sum cannot overflow into the sign bit.
          BO->setHasNoSignedWrap(false);
          errs() << "RemoveRedundantNSW: removed nsw from: " << *BO << "\n";
          Changed = true;
        }
      }
    }

    if (!Changed)
      return PreservedAnalyses::all();

    // We only changed instruction flags, not the CFG or def-use graph.
    PreservedAnalyses PA;
    PA.preserveSet<CFGAnalyses>();
    PA.preserve<DominatorTreeAnalysis>();
    PA.preserve<AssumptionAnalysis>();
    return PA;
  }
};

} // anonymous namespace

// Pass registration entry point for opt plugins.
extern "C" ::llvm::PassPluginLibraryInfo LLVM_ATTRIBUTE_WEAK
llvmGetPassPluginInfo() {
  return {
    LLVM_PLUGIN_API_VERSION,
    "RemoveRedundantNSW",
    "0.1",
    [](PassBuilder &PB) {
      PB.registerPipelineParsingCallback(
        [](StringRef Name, FunctionPassManager &FPM,
           ArrayRef<PassBuilder::PipelineElement>) {
          if (Name == "remove-redundant-nsw") {
            FPM.addPass(RemoveRedundantNSW());
            return true;
          }
          return false;
        });
    }
  };
}
```

### 198.8.3 Test IR and Expected Output

```llvm
; test.ll
define i32 @add_no_overflow(i32 %a, i32 %b) {
entry:
  ; Both %a and %b are masked to [0, 63] — top two bits zero.
  ; The nsw on the add is redundant because their sum fits in 7 bits,
  ; well within [0, INT_MAX].
  %ma = and i32 %a, 63           ; %ma in [0, 63], bits 6-31 known zero
  %mb = and i32 %b, 63           ; %mb in [0, 63], bits 6-31 known zero
  %sum = add nsw i32 %ma, %mb   ; nsw is redundant — remove it
  ret i32 %sum
}

define i32 @add_cannot_prove(i32 %a, i32 %b) {
  ; Only bit 31 is known zero (non-negative), not bit 30.
  ; Their sum could reach 2^31 - 2 + 2^31 - 2 = 2^32 - 4, overflowing i32.
  ; The pass must NOT remove nsw here.
  %ma = and i32 %a, 2147483647   ; bit 31 known zero, bits 0-30 unknown
  %mb = and i32 %b, 2147483647   ; bit 31 known zero
  %sum = add nsw i32 %ma, %mb   ; nsw NOT redundant — leave it
  ret i32 %sum
}
```

Running:

```bash
opt-22 -load-pass-plugin=./ValueTrackingDemo.so \
        -passes=remove-redundant-nsw \
        test.ll -S -o -
```

Expected output on stderr (only the first function triggers the pass):

```
RemoveRedundantNSW: removed nsw from:   %sum = add i32 %ma, %mb
```

And in the output IR for `@add_no_overflow`, `add nsw i32 %ma, %mb` becomes `add i32 %ma, %mb`. The `@add_cannot_prove` function is left unchanged.

### 198.8.4 Extending the Pass: Using ConstantRange

The `KnownBits` check above is conservative. A more precise check uses `ConstantRange`:

```cpp
// In the loop body, after getting KnownLHS and KnownRHS:
ConstantRange CRLHS = ConstantRange::fromKnownBits(KnownLHS, /*IsSigned=*/true);
ConstantRange CRRHS = ConstantRange::fromKnownBits(KnownRHS, /*IsSigned=*/true);

// Check: can the signed sum overflow?
// addWithNoWrap with NSW returns the range of (LHS + RHS) assuming no wrap.
// If that range equals the unconstrained add range, no wrap is possible.
ConstantRange SumNW  = CRLHS.addWithNoWrap(CRRHS,
                         OverflowingBinaryOperator::NoSignedWrap);
ConstantRange SumWrap = CRLHS.add(CRRHS);

if (SumNW == SumWrap) {
  // The NSW-constrained range equals the unrestricted range:
  // overflow cannot occur for any value in these ranges.
  BO->setHasNoSignedWrap(false);
  Changed = true;
}
```

`ConstantRange::fromKnownBits` converts the `KnownBits` lattice element into the corresponding `ConstantRange`. For `%ma = and i32 %a, 63`, this produces `ConstantRange([0, 64))`. The `addWithNoWrap(NoSignedWrap)` call computes `[0, 128)` restricted to the no-wrap domain. `add` computes the same `[0, 128)`. Since they match, the `nsw` is provably redundant. For the second function, `fromKnownBits` produces `ConstantRange([0, 2^31))`, and `addWithNoWrap` returns a proper subset of `add`'s result — a mismatch, so the pass correctly skips it.

---

## 198.9 Validation with Alive2

### 198.9.1 Why Formal Validation Matters

Value-tracking transformations operate on approximations — the `KnownBits` lattice is sound but not complete. A transformation justified by a `KnownBits` result must be proved sound in general: the result must hold for *all* possible inputs that satisfy the `KnownBits` precondition, not just the cases the developer tested. Alive2 provides this guarantee automatically. See [Ch170 — Alive2 and Translation Validation](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md) for a full treatment of the Alive2 framework.

### 198.9.2 The alive-tv Tool

`alive-tv` takes two LLVM IR functions — source and target — and checks that the target refines the source for all inputs. The canonical invocation:

```bash
alive-tv source.ll target.ll
```

The tool exhaustively checks (using SMT via Z3) that every value the target function can produce is also a valid result of the source function under the same inputs. A successful check reports:

```
Transformation seems to be correct!
```

A failed check reports a concrete counterexample with input values, for example:

```
ERROR: Source is more defined than target

Example:
  i32 %x = #x80000001 (2147483649, -2147483647)
  ...
Source value: #x7fffffff (2147483647)
Target value: poison
```

This output means the rewrite introduced poison where the source produced a defined value. The counterexample input `%x = 0x80000001` is the exact value to reproduce the failure in a unit test.

`alive-tv` is not installed system-wide in the LLVM binary distribution. It must be built from the Alive2 repository (`github.com/AliveToolkit/alive2`) with a Z3 solver available. In a CI environment, the standard pattern is to build Alive2 as a CMake external project and invoke it from the LLVM test suite via `llvm-lit` with the `ALIVE2_TV` substitution variable.

### 198.9.3 Verifying a DemandedBits-Enabled Peephole

The peephole `(x | 0xFF00) & 0xFF → x & 0xFF` (Section 198.4.2) is universally valid. We verify it with Alive2:

```llvm
; source.ll
define i32 @src(i32 %x) {
  %t1 = or  i32 %x, 65280    ; 0xFF00
  %t2 = and i32 %t1, 255     ; 0x00FF
  ret i32 %t2
}
```

```llvm
; target.ll
define i32 @tgt(i32 %x) {
  %t2 = and i32 %x, 255      ; 0x00FF
  ret i32 %t2
}
```

The argument: `(x | 0xFF00) & 0x00FF`. The `or` sets bits 8–15; `& 0x00FF` then zeros all bits above 7. The `0xFF00` contribution is completely masked away. Therefore `(x | 0xFF00) & 0xFF == x & 0xFF` for all `x`.

When `alive-tv source.ll target.ll` is invoked against these two functions, it returns:

```
Transformation seems to be correct!
```

No precondition on `%x` is needed — the transformation is unconditional.

### 198.9.4 Verifying a KnownBits-Conditional Peephole

Some KnownBits-driven rewrites require a precondition. The Alive2 precondition syntax uses `%cond = freeze` and `pre:` blocks. For example, to verify `(x & 0xFF) >> 4 == x >> 4` when `x` has bits 8–31 known zero:

```llvm
; source-conditional.ll
; Pre: x is in [0, 255] (bits 8-31 zero, encoded as %x = and i32 %raw, 255)
define i32 @src(i32 %raw) {
  %x    = and  i32 %raw, 255
  %mask = and  i32 %x, 255
  %r    = lshr i32 %mask, 4
  ret i32 %r
}
```

```llvm
; target-conditional.ll
define i32 @tgt(i32 %raw) {
  %x = and  i32 %raw, 255
  %r = lshr i32 %x, 4
  ret i32 %r
}
```

Both representations include the `and 255` explicitly, encoding the precondition structurally rather than as a separate `pre:` block. Alive2 checks the equivalence and confirms it.

### 198.9.5 Integration with the LLVM Test Suite

LLVM's `test/Transforms/InstCombine/` directory contains hundreds of `.ll` files that serve as regression tests for value-tracking-driven simplifications. Each test pairs an `; CHECK` line with the expected simplified IR, run under `FileCheck`. A typical test structure:

```llvm
; RUN: opt-22 -passes=instcombine -S < %s | FileCheck %s

; Test: (x & 0xFF) | 0x100 has bit 8 known-one — icmp ult with 256 folds to false
define i1 @test_knownbits_icmp(i32 %x) {
; CHECK-LABEL: @test_knownbits_icmp(
; CHECK-NEXT:    ret i1 false
  %masked = and i32 %x, 255
  %ored   = or  i32 %masked, 256
  %cmp    = icmp ult i32 %ored, 256
  ret i1 %cmp
}
```

The `; CHECK-NEXT: ret i1 false` asserts that InstCombine folds the compare to a constant via `computeKnownBits`. Running `llvm-lit test/Transforms/InstCombine/known-bits.ll` confirms the test.

Before contributing a new value-tracking optimization, the recommended workflow is:

1. Add a test in `test/Transforms/InstCombine/` or `test/Transforms/CorrelatedValuePropagation/` with both a positive case (the transform fires) and a negative case (a near-miss that should *not* fire, with `; CHECK: <original instruction>`).
2. Verify the test passes with `llvm-lit`.
3. If the peephole has any precondition (e.g., requires NUW or a known-bits constraint), construct the source and target IR as separate files and verify with `alive-tv` or the Alive2 web interface at `https://alive2.llvm.org/ce/`.
4. When submitting the patch, include the Alive2 output or web UI permalink in the commit message as evidence of soundness.

LLVM's code review culture for InstCombine patches requires either a proof sketch or an Alive2 confirmation for any non-trivial rewrite. The project has historical bugs from rewrites that looked correct but failed on unusual inputs (e.g., `INT_MIN` edge cases in signed arithmetic, or poison propagation differences). Alive2 catches these automatically.

---

## 198.10 Summary

- **ValueTracking** (`llvm/Analysis/ValueTracking.h`) is a stateless library of ~80 free functions that answer compile-time predicate questions about `Value*` objects, recursing up to `MaxAnalysisRecursionDepth = 6` levels through the def-use graph.

- **KnownBits** (`llvm/Support/KnownBits.h`) represents per-bit knowledge as two `APInt` fields (`Zero`, `One`) with invariant `Zero & One == 0`. `computeKnownBits(V, SQ)` propagates through all integer operations. `ComputeNumSignBits`, `isKnownNonZero`, `isKnownNonNegative`, and `isKnownPositive` are convenience wrappers used heavily in InstCombine.

- **ConstantRange** (`llvm/IR/ConstantRange.h`) represents integer value sets as half-open intervals `[Lower, Upper)` with wrap-around semantics. All arithmetic operations are supported; `addWithNoWrap` and `makeGuaranteedNoWrapRegion` enable precise no-overflow reasoning for ScalarEvolution and loop canonicalization.

- **DemandedBits** (`llvm/Analysis/DemandedBits.h`) runs a backward dataflow that computes, per instruction, which output bits are consumed by any user. InstCombine and CodeGenPrepare use it to eliminate operations whose outputs are entirely masked away by downstream uses.

- **LazyValueInfo** (`llvm/Analysis/LazyValueInfo.h`) provides demand-driven, flow-sensitive value inference by exploiting branch guards and `@llvm.assume` constraints. JumpThreading and CorrelatedValuePropagation are its primary consumers, using `getConstantRange` and `getPredicateAt` to fold branches and demote signed operations to unsigned equivalents.

- **SimplifyQuery** (`llvm/Analysis/SimplifyQuery.h`) bundles `DataLayout`, `TargetLibraryInfo`, `DominatorTree`, `AssumptionCache`, and a context instruction into a single struct passed to all value-tracking functions. The `AssumptionCache` field enables `@llvm.assume` intrinsics to contribute known bits and range constraints.

- **Pointer alignment** is tracked via `Value::getPointerAlignment(DL)` and the value-tracking-enhanced `getKnownAlignment` from `llvm/Transforms/Utils/Local.h`. Trailing known-zero bits in a pointer value imply alignment, feeding directly into vectorization alignment decisions.

- **Pass authoring**: request analyses via `FunctionAnalysisManager`, build a `SimplifyQuery` with a context instruction, call `computeKnownBits`, and test the resulting `KnownBits` fields. Removing flags (like `nsw`) is always sound; adding them requires a formal soundness proof.

- **Alive2** (`alive-tv`) formally verifies value-tracking-driven peepholes by SMT-checking that a target IR function refines its source for all inputs. It is the authoritative tool for confirming that a new optimization is correct before committing it to the LLVM tree.
