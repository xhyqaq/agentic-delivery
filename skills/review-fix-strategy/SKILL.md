---
name: review-fix-strategy
description: "Use when a reviewer returns issues after code review to decide the correct fix approach — routes between lightweight patch, full reimplementation, or escalation based on issue severity"
---

# Review Fix Strategy

Decide HOW to fix reviewer-reported issues based on severity. Not all problems deserve the same fix approach.

## When to Use

- Called by `agentic-delivery` at Stage 6 when any reviewer (Spec Compliance or Code Quality) returns issues
- The orchestrator (main agent) evaluates severity and routes accordingly

## Decision Flow

```
Reviewer returns issues
       │
       ▼
  Orchestrator assesses severity
       │
       ├── Minor / Important
       │     → Dispatch new Implementer subagent
       │     → Input: review report + relevant code files
       │     → After fix: re-review (new reviewer instance)
       │
       ├── Critical
       │     → Choose one:
       │       ├─ Option A: Dispatch independent Fix subagent
       │       │   Input: review report + original spec (NO original code)
       │       │   Reimplement from scratch to avoid inheriting flawed approach
       │       ├─ Option B: Roll back to plan stage, redesign this task
       │       └─ Option C: Escalate to user for decision
       │
       └── 3+ rounds failed
             → Force escalate to user
             → Attach: all review reports + fix attempt summaries
```

## Severity Classification

### Minor / Important

**Definition:** Issues that can be fixed by patching the existing code.

**Examples:**
- Logic bugs in edge cases
- Poor naming / magic numbers
- Missing boundary checks
- Missing or wrong comments
- Style violations
- Missing test cases

**Why patch is safe:** The review report precisely identifies the problem. Since Claude Code's Task tool is stateless, the new Implementer subagent is a fresh instance with no mental inertia from the original implementation. It receives the review report + code and fixes surgically.

**Fix subagent input:**
- Review report (full text)
- Relevant source files from the latest checkpoint (only the files mentioned in review)
- Task description (brief context)

When using checkpoint commits, patch fixes should start from the latest checkpoint rather than uncommitted workspace state.

### Critical

**Definition:** Issues that indicate the implementation approach itself is fundamentally wrong.

**Examples:**
- Violates the design spec's architecture
- Wrong algorithm or data structure choice
- Implementation solves the wrong problem
- Structural defect that can't be patched (need rewrite)
- Security flaw in the design approach

**Why patch is dangerous:** Fixing on a flawed foundation inherits the flawed thinking. The code structure itself guides the fix subagent toward the same mistakes.

**Option A — Fresh reimplementation:**
- Dispatch independent Fix subagent
- Give it: review report + original design spec
- Do NOT give it the original code
- Let it implement from scratch based on spec

**Option B — Redesign task:**
- Return to Stage 4 (plan writing)
- Redesign this specific task
- Re-dispatch with new plan

**Option C — User escalation:**
- When orchestrator cannot determine the right approach
- Present: the issue, what was tried, options considered

## Safety Net

### Re-review is mandatory

After ANY fix (Minor, Important, or Critical), a NEW reviewer subagent MUST review the result. This is non-negotiable.

- The re-reviewer is a fresh instance (not the original reviewer)
- This prevents "rubber-stamping" by a reviewer who already saw the code

### Max rounds

- **3 rounds per review stage** (Spec Compliance and Code Quality counted separately)
- After 3 failed rounds → force escalate to user
- Escalation includes: all review reports from all rounds + all fix attempt summaries

### Escalation format

```markdown
## Review Escalation

**Task:** [task name]
**Review Stage:** Spec Compliance / Code Quality
**Rounds attempted:** 3

### Round 1
- **Issues found:** [summary]
- **Fix attempted:** [summary]

### Round 2
- **Issues found:** [summary]
- **Fix attempted:** [summary]

### Round 3
- **Issues found:** [summary]
- **Fix attempted:** [summary]

### Recommendation
[Orchestrator's assessment of what's going wrong and suggested path forward]
```

## What NOT to Do

- Do NOT always use the same fix approach — severity determines strategy
- Do NOT skip re-review after fix — ever
- Do NOT let Critical issues be patched — reimplementation or redesign
- Do NOT exceed 3 rounds without escalating — infinite loops waste resources
- Do NOT give original code to Critical fix subagent — it biases toward the same mistakes

## Lifecycle

**This skill is a decision framework**, not a subagent. The orchestrator (main agent) applies this logic to decide which subagent to dispatch and with what context.
