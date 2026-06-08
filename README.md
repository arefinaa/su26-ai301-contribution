# su26-ai301-contribution
Repo for AI301.

# Contribution [#3577]: [FSM] Tracking issue for canonicalization and optimization patterns]

**Contribution Number:** [1 / 2 / 3]  
**Student:** Arefin Azam  
**Issue:** [https://github.com/llvm/circt/issues/3577&sa=D&source=editors&ust=1780962362869744&usg=AOvVaw3yIuToiXmhxnAVDHxSBi6W](https://github.com/llvm/circt/issues/3577]
**Status:** [Phase I Complete]

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
