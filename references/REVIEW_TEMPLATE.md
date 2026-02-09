# Peer Review Template

Use this template when reviewing a plan/spec document.

---

## Review Header

```markdown
# Peer Review: [Plan Title]

**Reviewer:** [Agent Name]
**Review Round:** [1 or 2]
**Date:** [ISO date]
**Plan Version:** [hash or version if available]

## Executive Summary

[2-3 sentence summary of the plan's purpose and scope]

## Overall Assessment

**GO / CONDITIONAL GO / NO-GO**

[1-2 sentence justification for the assessment]
```

## Findings Section

Each finding should follow this structure:

```markdown
## Findings

### [F1] [Finding Title]

**Severity:** Critical | Major | Minor | Suggestion
**Category:** Technical | Scope | Risk | Clarity | Dependencies | Testing

**Observation:**
[What you observed in the plan]

**Concern:**
[Why this is a problem]

**Recommendation:**
[Specific actionable recommendation]

---

### [F2] [Next Finding Title]
...
```

## Severity Definitions

| Severity | Definition | Impact on GO/NO-GO |
|----------|------------|-------------------|
| **Critical** | Fundamental flaw that will cause failure | Requires NO-GO |
| **Major** | Significant issue affecting success likelihood | Requires NO-GO or strong mitigation |
| **Minor** | Issue that should be addressed but won't block | Can proceed with CONDITIONAL GO |
| **Suggestion** | Enhancement idea, not a defect | No impact on assessment |

## Category Definitions

| Category | What to Look For |
|----------|------------------|
| **Technical** | Architecture, implementation approach, technology choices |
| **Scope** | Missing requirements, scope creep, unclear boundaries |
| **Risk** | Unmitigated risks, single points of failure, assumptions |
| **Clarity** | Ambiguous requirements, undefined terms, missing details |
| **Dependencies** | External dependencies, sequencing issues, blockers |
| **Testing** | Validation approach, acceptance criteria, test coverage |

## Strengths Section

```markdown
## Strengths

- [S1] [Positive aspect of the plan]
- [S2] [Another positive aspect]
```

## Questions Section

```markdown
## Questions for Author

- [Q1] [Clarifying question that affects assessment]
- [Q2] [Another question]
```

## Round 2 Specific Guidance

In Round 2, focus on:

1. **Resolution Assessment**: Were Round 1 concerns adequately addressed?
2. **New Issues**: Any issues introduced by revisions?
3. **Residual Risk**: What risks remain even after revisions?
4. **Implementation Readiness**: Is the plan now actionable?

```markdown
## Round 1 Concern Resolution

| Finding | Status | Notes |
|---------|--------|-------|
| F1 | Resolved / Partially Resolved / Unresolved | [Brief note] |
| F2 | ... | ... |
```

## Closing

```markdown
## Recommendation

[Final paragraph summarizing the review and recommendation for the Editor (human)]

**Final Assessment: GO / CONDITIONAL GO / NO-GO**
```
