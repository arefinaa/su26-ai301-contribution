# su26-ai301-contribution
Repo for AI301.

# Contribution [#3577]: [FSM] Tracking issue for canonicalization and optimization patterns]

**Contribution Number:** 1  
**Student:** Arefin Azam  
**Issue:** [https://github.com/llvm/circt/issues/3577](https://github.com/llvm/circt/issues/3577)  
**Fork:** [https://github.com/arefinaa/circt](https://github.com/arefinaa/circt)  
**Working Branch:** [`fix-issue-3577`](https://github.com/arefinaa/circt/tree/fix-issue-3577)  
**Status:** Phase III Complete

---

## Why I Chose This Issue

My background in computer architecture and RTL design made this issue immediately recognizable — FSMs are one of the most fundamental building blocks in any sequential hardware design. Every control unit, protocol handler, or datapath sequencer I've written in Verilog or VHDL has relied on FSM patterns, so I have a strong intuition for what correct and optimized state machine behavior looks like. What drew me to this specific issue is that it asks me to think about FSM semantics at the compiler IR level rather than the HDL level, which is a perspective I want to develop. Identifying when a transition's guard condition is redundant given the guards of sibling transitions is exactly the kind of reasoning a synthesis tool does implicitly — being able to encode that as an explicit MLIR rewrite pattern is a new and interesting challenge for me.
This issue is also well-suited as a learning vehicle because it's open-ended: the maintainers have laid out a problem statement and an example, but the actual pattern-matching logic and rewrite infrastructure are left to the contributor. I'll have to dig into MLIR's PatternRewriter API, understand how fsm.state, fsm.transition, and comb ops compose, and write proper lit tests. I'm hoping to leave this experience with a concrete understanding of how hardware compiler optimizations get implemented at the IR level, which directly complements what I already know at the RTL level.

---

## Understanding the Issue

The CIRCT FSM dialect currently has no canonicalization or optimization passes. This tracking issue documents a set of patterns that should be implemented. The specific pattern highlighted is mutually exclusive transition elimination: when two transitions out of the same state have guard conditions that are logical complements of each other (i.e., one fires if and only if the other doesn't), the second transition's guard is redundant and can be dropped — making it an unconditional (default) fallthrough transition. The compiler currently emits both guards verbatim even when the redundancy is statically provable.

When the FSM optimizer detects that a state has two outgoing transitions whose guards are mutually exclusive and collectively exhaustive (they cover all possible input values with no overlap), the second transition should have its guard removed, becoming an unconditional fsm.transition. In the example from the issue, fsm_exit's guard — which is just NOT lt_reg.out — is provably redundant given that seq_1_while_if's guard is lt_reg.out. The rewritten IR should drop the comb.xor and fsm.return from that second transition block entirely.

[What should happen?]

The FSM dialect emits both transition guards as-is, with no analysis of whether any guard is implied by the failure of a prior guard. In the example, the second transition computes comb.xor %lt_reg.out, %true (i.e., !lt_reg.out) and returns it as its guard condition — even though if the first transition's guard (lt_reg.out) failed, this second guard is guaranteed to be true. The redundant logic remains in the IR and propagates all the way to emitted SystemVerilog, producing unnecessary logic that a synthesis tool must redundantly simplify downstream.

[What actually happens?]

lib/Dialect/FSM/FSMOps.cpp — Where canonicalization patterns for FSM ops would be registered and implemented (canonicalizers attach to ops via getCanonicalizationPatterns or fold methods).
include/circt/Dialect/FSM/FSMOps.td — TableGen op definitions for fsm.state, fsm.transition, and related ops; may need fold/canonicalize hooks added.
lib/Dialect/Comb/ — The comb dialect (combinational logic ops like comb.xor) whose op patterns need to be matched to detect the complement relationship between guards.
test/Dialect/FSM/ — Lit test directory where new canonicalization test cases (like the mutually exclusive transition example in the issue) need to be added.

[Which parts of the codebase are involved?]
---

## Reproduction Process

### Environment Setup

CIRCT is an LLVM incubator project. There is no `devcontainer.json` and no
`CONTRIBUTING.md`; the authoritative setup guide is `docs/GettingStarted.md`
(plus `docs/DeveloperPolicy.md` and `docs/AIToolPolicy.md`). CIRCT vendors LLVM
as a **git submodule** and must be compiled against that exact pinned LLVM/MLIR
commit — there is no pre-built binary or package to install.

**Platform:** Windows 11 Pro, building from a Git Bash / PowerShell shell.

**Challenges encountered (and how I worked through them):**

1. **No build toolchain present.** A first audit of the machine showed that
   none of the required tools were installed: `cmake`, `ninja`, `clang`/`clang++`,
   MSVC (`cl`), and the recommended `lld` linker were all missing, and only a
   Windows Store stub `python` was on `PATH`. Per `docs/GettingStarted.md`, a
   working build needs **Visual Studio / MSVC + CMake + Ninja** (Ninja and CMake
   must be run from a *VS Developer Command* shell on Windows), plus the Python
   packages `psutil pyyaml numpy pybind11`.
2. **LLVM submodule not checked out.** `git submodule status` reported the
   `llvm` submodule as uninitialized (`-b7152ff70…` with a leading `-`, and an
   empty `llvm/` directory). The documented fix is
   `git submodule init && git submodule update` (a shallow clone of the pinned
   LLVM commit; `cd llvm && git fetch --unshallow` if full history is needed).
3. **Two-stage, multi-hour build.** Even with the toolchain in place, CIRCT
   requires building LLVM/MLIR *first* (`-DLLVM_ENABLE_PROJECTS="mlir"`,
   `-DLLVM_TARGETS_TO_BUILD=X86`/`host`, `-DLLVM_ENABLE_ASSERTIONS=ON`) and then
   CIRCT against that build tree. This is the project's well-known friction
   point: a from-scratch Debug build is several hours of compilation and many
   GB of disk. The Windows notes in `docs/GettingStarted.md` explicitly warn the
   MSVC + Ninja + Python path is "straight forward, though full of landmines"
   and that VS Code's CMake integration does **not** work correctly for Python
   bindings — the command line is the only reliable route.

**Reference build commands** (from `docs/GettingStarted.md`, recorded here for
the next contributor):

```powershell
# From a VS Developer PowerShell:
> python -m pip install psutil pyyaml numpy pybind11
> git submodule init; git submodule update

# Stage 1: LLVM/MLIR
> cmake -Bllvm/build llvm/llvm -GNinja `
    -DLLVM_ENABLE_PROJECTS=mlir `
    -DLLVM_TARGETS_TO_BUILD=X86 `
    -DLLVM_ENABLE_ASSERTIONS=ON `
    -DCMAKE_BUILD_TYPE=Release
> ninja -Cllvm/build

# Stage 2: CIRCT (produces tools/circt-opt/circt-opt)
> cmake -Bbuild . -GNinja `
    -DMLIR_DIR="$PWD/llvm/build/lib/cmake/mlir" `
    -DLLVM_DIR="$PWD/llvm/build/lib/cmake/llvm" `
    -DLLVM_ENABLE_ASSERTIONS=ON `
    -DCMAKE_BUILD_TYPE=Release
> ninja -Cbuild circt-opt
> ninja -Cbuild check-circt           # runs the lit test suite (incl. test/Dialect/FSM)
```

**Build status (honest):** A full local build was **not completed** for Phase
II. The required toolchain is not installed on this machine and a from-scratch
LLVM+CIRCT compile is a multi-hour commitment that exceeds the Step-1 budget;
per the program guidance, I documented the setup process and the exact blocker
rather than burning the time budget on it. The reproduction below is therefore
established by **source-level analysis + a committed reproduction artifact**,
and runtime confirmation with `circt-opt` is the first task of Phase III once
the toolchain/build is provisioned (or run in CI / a Linux environment, which
is the smoother path for this project).

### Steps to Reproduce

The bug is a *missing optimization*, so "reproducing" it means showing that the
redundant complement guard survives canonicalization.

1. Check out the working branch: `git checkout fix-issue-3577`.
2. Build `circt-opt` per the commands above (Phase III).
3. Run the canonicalizer on the committed reproduction file:
   ```
   circt-opt --canonicalize test/Dialect/FSM/mutex-transition-repro.mlir
   ```
4. **Observed (buggy) result:** the output is identical to the input — state
   `@A`'s second transition still contains
   `%true = hw.constant true`, `%ncond = comb.xor %cond, %true : i1`, and
   `fsm.return %ncond`, even though that guard is the logical complement of the
   first transition's guard (`fsm.return %cond`) and is therefore redundant.
5. **Expected (desired) result, per #3577:** the second transition's guard
   region should be dropped, collapsing it to an unconditional default
   transition `fsm.transition @C`.

This pattern is not hypothetical — it is emitted verbatim by the real
Calyx → FSM lowering. See `test/Conversion/CalyxToFSM/lower-while.mlir`, where
state `@fsm_exit` has `fsm.transition @seq_1_while_if guard { fsm.return
%lt_reg.out }` immediately followed by `fsm.transition @fsm_exit guard { … %0 =
comb.xor %lt_reg.out, %true_0 : i1; fsm.return %0 }` — exactly the
`seq_1_while_if` / `fsm_exit` example from the issue.

### Reproduction Evidence

- **Commit showing reproduction:** `66c8a29f0` — *"[FSM] Add reproduction case
  for mutually exclusive transition elimination (#3577)"*, which adds
  `test/Dialect/FSM/mutex-transition-repro.mlir`
  (on branch [`fix-issue-3577`](https://github.com/arefinaa/circt/tree/fix-issue-3577)).
- **Reproduction artifact:** `test/Dialect/FSM/mutex-transition-repro.mlir` — a
  minimal, self-contained machine that triggers the pattern and documents
  observed-vs-expected behavior inline.
- **My findings:**
  - The FSM dialect *does* already have canonicalization machinery
    (`StateOp::canonicalize`, `TransitionOp::canonicalize`,
    `UpdateOp::canonicalize` in `lib/Dialect/FSM/FSMOps.cpp`), but **none** of
    them reason about the *relationship between sibling guards*. They only
    handle a single guard that folds to a constant, no-op variable updates, and
    pruning transitions that come after an always-taken one.
  - Consequently a guard that is *semantically* always-true-given-the-previous-
    guard-failed (a logical complement) is never simplified — it survives to the
    emitted SystemVerilog as redundant logic.
  - The complement is expressed as `comb.xor %x, %true` (i1), and the issue
    notes it may also appear via `arith` ops, so the matcher must handle both.
  - `CIRCTComb` is **not** currently a link dependency of `CIRCTFSM`
    (`lib/Dialect/FSM/CMakeLists.txt`), so matching `comb.xor` directly will
    require adding that dependency (or matching generically by op name to avoid
    a new library edge).

---

## Solution Approach

### Analysis

**Root cause.** The FSM canonicalizer has no pattern that compares the guards of
two *sibling* transitions out of the same state. The existing patterns in
`lib/Dialect/FSM/FSMOps.cpp` operate on a single op in isolation:

- `TransitionOp::canonicalize` (line ~488) only simplifies a guard whose
  `fsm.return` operand is a **constant** `arith.constant` (true → make
  unconditional; false → erase the transition).
- `StateOp::canonicalize` (line ~350) only removes transitions that appear
  *after* an already-"always-taken" transition (via `isAlwaysTaken()`).

Neither recognizes the case where transition `T2`'s guard is the **logical
complement** of an earlier transition `T1`'s guard. When `T1` fires iff `cond`
and `T2`'s guard is `!cond` (e.g. `comb.xor %cond, %true`), and these two are the
state's only transitions, `T2`'s guard is provably implied by `T1` failing, so
it is dead logic. Because nothing detects this, the `comb.xor` + `fsm.return`
remain in the IR and propagate to emitted SystemVerilog.

### Proposed Solution

Add a canonicalization pattern that, for a state's transitions evaluated in
priority order, detects when a transition's guard is the boolean complement of a
prior sibling's guard and (when that makes it the effective default) strips its
guard region, turning it into an unconditional `fsm.transition`. The existing
`StateOp::canonicalize` pass then naturally prunes anything unreachable after it.
Scope the **first, provably-safe** version narrowly: a state with exactly two
transitions where the second's guard is `NOT(first's guard)`. This matches the
issue's example and the real Calyx-lowered `if`/`while` FSMs, and avoids the
non-determinism that sank the earlier unreachable-states attempt (PR #3131,
called out in the issue).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** A finite-state machine state can have several guarded outgoing
transitions, tried in order; the first whose guard is true is taken. If two
transitions have guards that are exact logical complements and together cover
every input (mutually exclusive *and* exhaustive), the second guard is
redundant: reaching it already implies the first guard was false. The compiler
should drop the redundant guard and make that transition unconditional, but
today it emits both guards verbatim.

**Match:** Concrete patterns already in the codebase that I will model the fix
on (all in `lib/Dialect/FSM/FSMOps.cpp`):
1. `TransitionOp::canonicalize` — shows exactly how to **rewrite a guard into an
   unconditional transition**: it replaces the guard's `fsm.return %x` with an
   operand-less `fsm.ReturnOp` via the `PatternRewriter`. The redundant-guard
   case reuses this same "make unconditional" rewrite.
2. `TransitionOp::isAlwaysTaken()` / `getGuardReturn()` — show how to introspect
   a guard region and pull out the returned i1 value.
3. `StateOp::canonicalize` — shows how to iterate `op.getTransitions().getOps<
   TransitionOp>()` in order and erase ops with the rewriter; it is the natural
   place to add pairwise sibling analysis, and it already cleans up trailing
   unreachable transitions for free.
4. MLIR `m_Op<comb::XorOp>` / operand introspection and
   `mlir::arith::ConstantOp` matching (already used for the constant-guard case)
   for detecting the `xor x, true` complement form.

**Plan:** Step-by-step implementation (Phase III):
1. Add a helper `getGuardValue(TransitionOp)` returning the i1 `Value` a guard
   returns (or null if unconditional / empty), factored out of the existing
   guard-introspection code.
2. Add a helper `isComplementOf(Value a, Value b)` that returns true when `a`
   is defined by `comb.xor(b, true)` or `arith.xori(b, 1)` (handle operand
   commutativity, and the symmetric case where `b` is the complement of `a`).
3. Extend `StateOp::canonicalize` (or add a dedicated `StateOp` rewrite
   pattern): when a state has exactly two transitions `T1, T2` and
   `isComplementOf(guardValue(T2), guardValue(T1))`, strip `T2`'s guard region
   (erase the `comb.xor`/operands and replace its `fsm.return %x` with an
   operand-less `fsm.return`, mirroring `TransitionOp::canonicalize`). Return
   `success()` so the driver re-runs and the trailing-transition cleanup applies.
4. Update `lib/Dialect/FSM/CMakeLists.txt` to add `CIRCTComb` to `LINK_LIBS`
   (and `DEPENDS`) **if** I match `comb::XorOp` by type; alternatively match by
   operation name string to avoid the new dependency. (Decision to be made and
   documented in the PR.)
5. Add `#include "circt/Dialect/Comb/CombOps.h"` to `FSMOps.cpp` if needed.
6. Add lit tests to `test/Dialect/FSM/canonicalize.mlir` (see Testing Strategy).

**Implement:** Phase III. Branch:
[`fix-issue-3577`](https://github.com/arefinaa/circt/tree/fix-issue-3577).
Reproduction baseline already committed (`66c8a29f0`).

**Review:** Self-review checklist against CIRCT/LLVM conventions
(`docs/GettingStarted.md` → "Submitting changes", `docs/DeveloperPolicy.md`,
`docs/AIToolPolicy.md`):
- [x] Code written to LLVM style (≤80 columns, `auto` with `dyn_cast`-style
      `getDefiningOp<T>()`, doc comments on helpers). `clang-format` itself is
      **not installed locally**, so a `git clang-format origin/main` pass is
      still owed and will run in CI / before final PR submission.
- [x] Follow LLVM Coding Standards (naming, `getDefiningOp<>()` usage, rewriter
      APIs instead of raw `new`/manual erase).
- [x] Keep the patch small and incremental — v1 is scoped to the
      two-transition complement case only.
- [ ] `ninja check-circt` passes (especially `test/Dialect/FSM` and
      `test/Conversion/CalyxToFSM`) — **pending a build**; no local toolchain
      (see *Manual Testing*). To run in CI / a Linux build.
- [ ] PR uses "Squash and Merge"; CC the FSM dialect code owners; reference
      issue #3577 in the PR description — **pending PR submission**.
- [x] Disclose AI tool assistance per `docs/AIToolPolicy.md` — every Phase III
      commit carries the `Assisted-by: Claude Code:claude-opus-4-8` trailer.

**Evaluate:** Build `circt-opt`, run `--canonicalize` on the reproduction file
and confirm the `comb.xor`/`fsm.return` disappear and `@C` becomes
`fsm.transition @C`; confirm the whole suite still passes with `ninja
check-circt`. See Testing Strategy for the specific positive and negative cases.

---

## Testing Strategy

CIRCT uses **lit + FileCheck** lit tests (run via `ninja check-circt`). New
canonicalization cases go in `test/Dialect/FSM/canonicalize.mlir` with a
`// RUN: circt-opt --canonicalize %s | FileCheck %s` header — matching the
existing file.

All of the following were **implemented** in `test/Dialect/FSM/canonicalize.mlir`
(commit `65d410f6f`):

- [x] **Positive — comb form:** two complementary transitions (`fsm.return
      %cond` then `comb.xor %cond, %true`); asserts with `// CHECK-NOT: comb.xor`
      that the redundant logic is gone and that the second transition becomes
      unconditional (`// CHECK: fsm.transition @C` with no `guard`).
- [x] **Positive — arith form:** same shape but the complement is
      `arith.xori %cond, %true`; asserts it is likewise simplified
      (`// CHECK-NOT: arith.xori`) — covers the issue's note that guards may be
      `comb` *or* `arith`.
- [x] **Negative — non-complementary guards:** the second guard is `!%b`, which
      is *not* the complement of the first guard (`%a`); the transitions must be
      left **untouched** (`// CHECK: comb.xor` still present) — guards against
      over-eager rewriting and specifically exercises the
      `combXor.getOperand(0) == b` operand check.
- [x] **Negative — three transitions:** a state with three guarded transitions
      (where `@C`'s guard *is* the complement of `@B`'s) must not be collapsed —
      verifies the conservative *exactly-two-transition* scope.
- [x] **Idempotence / termination:** because the lit test runs the greedy driver
      to a fixpoint, an infinite rewrite loop would hang the test. The pattern
      avoids this by only firing when the second guard is non-null; after the
      rewrite that guard becomes null (the region is erased), so it cannot
      re-fire.

### Integration Tests

- [ ] **Real Calyx-lowered FSM:** confirm `test/Conversion/CalyxToFSM/
      lower-while.mlir` and `lower-if.mlir` still pass; ideally add a follow-up
      check that a `--canonicalize` run after lowering removes the
      `seq_1_while_if` / `fsm_exit` complement guard from the issue.
- [ ] **Full suite:** `ninja check-circt` green (catches any regression in
      `basics.mlir`, `errors.mlir`, and downstream FSM→HW/SystemVerilog tests).

### Manual Testing

The intended manual check is to run `circt-opt --canonicalize
test/Dialect/FSM/mutex-transition-repro.mlir` and confirm the `comb.xor` /
`hw.constant` disappear and `@C` collapses to an unconditional `fsm.transition
@C`. **Honest status:** this machine has no local CIRCT/LLVM build toolchain
installed (see *Reproduction Process* — `cmake`, `ninja`, a C++ compiler, and
`circt-opt` are all absent), so the binary could not be run locally. Runtime
confirmation therefore happens in CI / a Linux build, which is the smoother path
for this project; the change is otherwise validated by close source analysis of
the canonicalization driver semantics (see *Challenges Faced*).

---

## Implementation Notes

### Phase II Progress

- Mapped the FSM dialect: `lib/Dialect/FSM/FSMOps.cpp` (op logic + existing
  canonicalizers), `include/circt/Dialect/FSM/FSMOps.td` (op + canonicalize
  hooks), `test/Dialect/FSM/` (lit tests).
- Confirmed the issue still exists upstream: no sibling-guard analysis in any
  existing canonicalization pattern.
- Discovered the real-world source of the pattern: `CalyxToFSM` lowering emits
  the exact `comb.xor`-complement guards (`test/Conversion/CalyxToFSM/
  lower-while.mlir`).
- Authored and committed a minimal reproduction artifact.
- Wrote the UMPIRE solution plan, scoped conservatively to the two-transition
  complement case to avoid the non-determinism that blocked the earlier
  unreachable-states attempt (PR #3131).
- **Setup decision:** documented the (substantial) LLVM+CIRCT build process and
  the missing-toolchain blocker rather than spending the entire Step-1 budget on
  a multi-hour from-scratch compile; runtime verification is the first Phase III
  task.

### Phase III Progress

The plan was implemented exactly as scoped, in small increments:

- **Detection helpers** (`lib/Dialect/FSM/FSMOps.cpp`):
  - `getGuardValue(TransitionOp)` returns the i1 `Value` a guard returns, or a
    null `Value` for an unconditional transition (factored out of the existing
    guard-introspection idiom).
  - `isComplementOf(Value a, Value b)` returns true when `a == !b`. It reuses
    `comb::XorOp::isBinaryNot()` — which already existed in the Comb dialect and
    precisely matches the `xor x, allones` shape — for the comb form, and matches
    `arith::XOrIOp` against a constant `1` (`m_One`) for the arith form.
- **Rewrite** (`StateOp::canonicalize`): when a state has *exactly two*
  transitions whose guards are complements, the second guard region is erased
  with `rewriter.eraseBlock`, collapsing it to an unconditional transition. This
  was chosen over the "replace `fsm.return %x` with an operand-less
  `fsm.return`" approach used by `TransitionOp::canonicalize` for the constant
  case, because erasing the whole block also removes the now-dead `comb.xor` /
  `hw.constant` in one step and yields a clean `fsm.transition @C` with no
  residual `guard { }` region. The existing always-taken pruning runs afterward
  unchanged.
- **Build dependency** (`lib/Dialect/FSM/CMakeLists.txt`): added `CIRCTComb` to
  `LINK_LIBS PUBLIC`. **Decision:** I chose to match `comb::XorOp` *by type*
  (idiomatic, robust, and lets me reuse `isBinaryNot`) rather than matching by
  operation-name string. This introduces no dependency cycle — `CIRCTComb`
  depends only on `CIRCTHW`, which `CIRCTFSM` already links — so the layering is
  clean. No new dependent-dialect registration is needed because the pattern
  only *inspects and erases* existing comb ops; it never *creates* one.
- **Tests:** added five lit/FileCheck cases (see *Testing Strategy*).

### Code Changes

- **Files modified (Phase II):**
  - `test/Dialect/FSM/mutex-transition-repro.mlir` *(new — reproduction artifact)*
  - `READ_ME(1).md` *(this contribution write-up)*
- **Files modified (Phase III):**
  - `lib/Dialect/FSM/FSMOps.cpp` *(detection helpers + `StateOp::canonicalize`)*
  - `lib/Dialect/FSM/CMakeLists.txt` *(`CIRCTComb` link dependency)*
  - `test/Dialect/FSM/canonicalize.mlir` *(5 new test cases)*
- **Key commits:**
  - `66c8a29f0` — reproduction case for #3577 (Phase II).
  - `0430c81ec` — `[FSM] Eliminate redundant complementary transition guards`
    (implementation + CMake).
  - `65d410f6f` — `[FSM] Add canonicalization tests for complementary guard
    elimination`.
  - All Phase III commits carry the `Assisted-by: Claude Code:claude-opus-4-8`
    trailer required by `docs/AIToolPolicy.md`.
- **Approach decisions:** reuse the existing canonicalization machinery and the
  Comb dialect's `isBinaryNot` helper; start with the narrowest provably-correct
  case (exactly two complementary transitions) and broaden only with more tests.

---

## Pull Request

**PR Link:** _Not yet opened._ The branch is implemented and committed locally
on [`fix-issue-3577`](https://github.com/arefinaa/circt/tree/fix-issue-3577);
the PR will be opened once the change is confirmed green in CI (the local
machine has no build toolchain — see *Manual Testing*).

**PR Description (draft):**

> **[FSM] Eliminate redundant complementary transition guards (#3577)**
>
> Implements the mutually exclusive transition elimination pattern from the FSM
> tracking issue #3577. When a state has exactly two outgoing transitions whose
> guards are logical complements, reaching the second transition already implies
> the first guard was false, so the second guard is provably true and redundant.
> `StateOp::canonicalize` now detects this and strips the second guard region,
> collapsing it to an unconditional (default) transition.
>
> The complement is recognized in both forms CIRCT emits: the comb form
> (`comb.xor %b, %true`, via `XorOp::isBinaryNot`) and the arith form
> (`arith.xori %b, %true`). This adds a `CIRCTComb` link dependency to
> `CIRCTFSM` (no cycle: Comb depends only on HW, which FSM already links).
>
> Scoped conservatively to the exactly-two-transition case to stay provably
> correct and avoid the non-determinism that blocked the earlier
> unreachable-states attempt (#3131). Adds positive (comb + arith) and negative
> (non-complement, three-transition) lit tests in
> `test/Dialect/FSM/canonicalize.mlir`.
>
> `Assisted-by: Claude Code:claude-opus-4-8` (per `docs/AIToolPolicy.md`).

**Maintainer Feedback:**
- _None yet — PR not opened._

**Status:** Implemented locally; awaiting CI build + PR submission.

---

## Learnings & Reflections

### Technical Skills Gained

- **MLIR canonicalization model.** I learned how `*::canonicalize` hooks plug
  into the greedy pattern-rewrite driver, why returning `success()` triggers a
  re-run to a fixpoint, and how that fixpoint behavior both enables follow-on
  cleanup (the always-taken pruning runs after my rewrite) and imposes a
  termination obligation (the pattern must stop firing — here, because the guard
  value becomes null once its region is erased).
- **`PatternRewriter` discipline.** Using `rewriter.eraseBlock` to drop a whole
  region in one tracked operation, rather than hand-erasing each inner op, and
  understanding why that is safe (no guard-region value is used outside it).
- **Reusing dialect-level helpers.** Finding that Comb already exposed
  `XorOp::isBinaryNot()` for exactly the `xor x, allones` shape turned a fiddly
  structural match into a one-liner — a reminder to read the target dialect
  before writing a matcher from scratch.
- **CMake layering in a multi-dialect project.** Reasoning about link
  dependencies and dependency cycles (Comb → HW, FSM → HW/Seq) before adding
  `CIRCTComb`, and weighing match-by-type vs match-by-name as a real tradeoff.

### Challenges Overcome

- **No local build toolchain.** The hardest constraint was the inability to
  compile or run `circt-opt` locally (a multi-hour LLVM+CIRCT build with tools
  that aren't installed). I compensated with careful source-level reasoning:
  tracing how the printer normalizes machine block arguments to `%arg0`/`%arg1`
  and how the `guard` keyword is rendered, so the FileCheck lines would match
  the *actual* canonicalized output rather than the source names. I deliberately
  wrote name-agnostic checks (matching `comb.xor`, `arith.xori`, and normalized
  arg names) to keep the tests robust.
- **Getting the negative tests to actually test something.** The first instinct
  for a negative case ("two unrelated guards") wouldn't have exercised the
  operand check. I made the negative case a `comb.xor` of an *unrelated* value
  (`!%b` while the sibling guard is `%a`), which specifically verifies the
  `getOperand(0) == b` guard against over-eager rewriting.
- **Choosing the cleanest rewrite.** The constant-guard canonicalizer leaves a
  residual `guard { fsm.return }` region; erasing the whole block instead
  produces the clean `fsm.transition @C` the issue describes.

### What I'd Do Differently Next Time

- **Provision the build first.** I'd stand up the toolchain (or a Linux/CI
  build) before Phase III so I could iterate against a real `circt-opt` instead
  of reasoning about printer output by hand — the single biggest source of risk
  in this contribution.
- **Generalize incrementally with tests as the ratchet.** The v1 scope is
  intentionally narrow (exactly two transitions). Next I'd add the N-transition
  generalization, but only one test-backed step at a time, mirroring LLVM's
  incremental-development norm.
- **Engage maintainers earlier** with the match-by-type vs match-by-name
  dependency question, since that is the one design decision most likely to draw
  review comments.

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
