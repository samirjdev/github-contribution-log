# Contribution [1]: [Heap Object Typing from VTables]

**Contribution Number:** 1
**Student:** [Samir J] 
**Issue:** [https://github.com/pwndbg/pwndbg/issues/179]  
**Status:** [Phase I] [Complete]

---

## Why I Chose This Issue

[This one sits right at the intersection of everything I do in security. I play CTFs and most of my  work depends on understanding *what* lives in an allocation — which object I'm corrupting, where its vtable points, whether I can pivot a UAF or type confusion into control flow. This issue is essentially building that reasoning into the tool: walk the heap, recognize vtable pointers, and infer object types from them. It also bridges directly into the blue-team and reverse-engineering side I care about , because identifying C++ objects in a live process or memory image is exactly what you do when reversing C++ malware or doing memory forensics. 

What I most want to learn is the layer beneath the exploitation I already know how to do. I can use vtable hijacking and RTTI in practice, but implementing type inference from scratch means really understanding how `type_info` hangs off `vtable[-1]`, how multiple inheritance lays out subobjects and offset-to-top, how RELRO and PIE move these structures around at runtime, and how to fall back to symbol resolution when RTTI is stripped. I also want experience working inside a real, mature codebase: reading pwndbg's heap-walking and vmmap internals, scoping a large feature into incremental PRs, and collaborating with maintainers on design decisions rather than just shipping code in isolation]

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

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
