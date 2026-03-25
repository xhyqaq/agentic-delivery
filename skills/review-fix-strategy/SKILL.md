---
name: review-fix-strategy
description: "Use when a reviewer returns issues after code review to decide the correct fix approach — routes between lightweight patch, full reimplementation, or escalation based on issue severity"
---

# Review Fix Strategy

Decide HOW to fix reviewer-reported issues based on severity. Not all problems deserve the same fix approach.

## When to Use

- Called by `agentic-delivery` at Stage 5 (before implementation) for Review Track Selection
- Called at Stage 6 when any reviewer returns issues to decide fix approach
- The orchestrator (main agent) evaluates task complexity and severity to route accordingly

## Decision Flow

### Part 1: Review Track Selection (Stage 5 - before implementation)

Analyze each task to determine appropriate review strategy based on complexity.

```
Task ready for implementation
       │
       ▼
  Analyze task characteristics
       │
       ├── Simple task (Fast Track)
       │     • Single file < 50 lines
       │     • No new API/interface
       │     • No architecture changes
       │     • Pure function/utility
       │     → Review: Code Quality ONLY
       │     → Estimated time: 1x
       │
       ├── Standard task (Standard Track)
       │     • Multiple files
       │     • New API/business logic
       │     • Typical feature work
       │     → Review: Spec Compliance → Code Quality
       │     → Estimated time: 2x
       │
       └── Complex task (Heavy Track)
             • Cross-domain integration (frontend + backend)
             • Architecture changes
             • Security-sensitive code
             • Complex multi-module coordination
             → Review: Spec Compliance → Code Quality → Integration
             → Estimated time: 3x
```

#### Track Selection Criteria

**Fast Track - Code Quality Only**

Conditions (ALL must be true):
- Single file modification < 50 lines
- No new public APIs or interfaces
- No architecture or design pattern changes
- Pure logic/utility implementation
- No cross-module dependencies

Rationale: Spec is clear and minimal. Code quality is the only risk.

**Standard Track - Spec + Code Quality**

Conditions (ANY is true):
- Multi-file changes
- New API definitions
- Business logic changes
- Database schema modifications
- Component interface changes

Rationale: Default path. Spec compliance ensures correctness, code quality ensures maintainability.

**Heavy Track - Spec + Code + Integration**

Conditions (ANY is true):
- Changes span frontend AND backend
- Architecture pattern changes (e.g., adding new middleware layer)
- Security-sensitive operations (auth, permissions, data encryption)
- Complex multi-module coordination
- External service integration

Rationale: Integration issues are high-risk. Extra verification prevents runtime failures.

#### Auto-Detection Algorithm

```
function selectReviewTrack(task):
    // Extract task metadata
    files = task.affected_files
    lines_changed = task.estimated_lines
    has_new_api = task.creates_api
    has_architecture_change = task.modifies_architecture
    domains = task.domains // ['frontend', 'backend', 'database']
    is_security_sensitive = task.security_sensitive

    // Heavy Track checks (highest priority)
    if (domains.length >= 2):
        return "Heavy" // Cross-domain
    if (has_architecture_change):
        return "Heavy" // Architecture change
    if (is_security_sensitive):
        return "Heavy" // Security-critical

    // Fast Track checks (only if very simple)
    if (files.length == 1 and
        lines_changed < 50 and
        not has_new_api and
        not has_architecture_change):
        return "Fast" // Simple utility

    // Default to Standard Track
    return "Standard"
```

#### Track Assignment Logging

When assigning tracks, log the decision:

```markdown
**Review Track Assignment:**

Task 1: Add validation helper → Fast Track
  - Single file (utils/validation.ts), 35 lines
  - No API changes
  - Pure utility function

Task 2: User authentication API → Standard Track
  - 3 files (routes, controller, service)
  - New public API
  - Business logic

Task 3: Payment integration → Heavy Track
  - Cross-domain (frontend checkout + backend payment service)
  - Security-sensitive (payment credentials)
  - External API integration
```

---

### Part 2: Fix Strategy Selection (Stage 6 - when review fails)

When reviewer returns issues, decide the fix approach.

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

**Fix Agent Dispatch (NEW - Unified Fix Subagent):**

When review fails, use the dedicated fix agent subagent (`subagent-driven-development/fix-agent-prompt.md`) for all severity levels:

**For Minor/Important issues:**
```
Task tool:
  description: "Fix review issues"
  prompt: [content from fix-agent-prompt.md]
  input:
    - review_report: [full JSON from reviewer]
    - relevant_files: [source files mentioned in review, from checkpoint commit]
    - checkpoint_commit: [SHA of implementer's checkpoint]
    - severity: "Minor" | "Important"
    - task_description: [brief context]
    - working_directory: [project root]
  subagent_type: "general-purpose"
```

**For Critical issues:**
```
Task tool:
  description: "Fix critical review issues"
  prompt: [content from fix-agent-prompt.md]
  input:
    - review_report: [full JSON from reviewer]
    - design_spec: [original requirements from design-spec.md - excerpt relevant to this task]
    - severity: "Critical"
    - task_description: [brief context]
    - working_directory: [project root]
    - NO original code (to avoid bias toward flawed approach)
  subagent_type: "general-purpose"
```

**Fix agent lifecycle:**
1. Orchestrator dispatches fix agent with severity-appropriate context
2. Fix agent analyzes issues and implements fixes
3. Fix agent creates fix commit
4. Fix agent returns: `DONE` (with fix commit SHA) or `BLOCKED` (needs user decision)
5. Orchestrator closes fix agent immediately
6. If `DONE`: dispatch new reviewer for re-review
7. If `BLOCKED`: escalate to user with fix agent's questions

See `subagent-driven-development/fix-agent-prompt.md` for complete fix agent specification.

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

- **Max rounds per review stage** depends on track:
  - Fast Track: 1 round (simple tasks should pass quickly)
  - Standard Track: 2 rounds
  - Heavy Track: 3 rounds (complexity warrants more attempts)
- After max rounds exceeded → force escalate to user
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

- Do NOT use the same review track for all tasks — analyze complexity first
- Do NOT skip track selection at Stage 5 — it optimizes review efficiency
- Do NOT override track selection without good reason — trust the criteria
- Do NOT always use the same fix approach — severity determines strategy
- Do NOT skip re-review after fix — ever
- Do NOT let Critical issues be patched — reimplementation or redesign
- Do NOT exceed max rounds without escalating — infinite loops waste resources
- Do NOT give original code to Critical fix subagent — it biases toward the same mistakes

## Performance Impact

**Example: 10-task feature**

Without smart routing (all Standard Track):
- 10 tasks × 2 reviews = 20 review calls
- Total time: ~20 minutes

With smart routing:
- 3 Fast Track tasks × 1 review = 3 calls
- 6 Standard Track tasks × 2 reviews = 12 calls
- 1 Heavy Track task × 3 reviews = 3 calls
- Total: 18 review calls (~10% reduction)
- Time saved: ~2 minutes + reduced fix rounds

**Key benefit:** Fast Track tasks fail faster (1 round vs 2), reducing wasted fix attempts.

## Lifecycle

**This skill is a decision framework**, not a subagent. The orchestrator (main agent) applies this logic to decide which subagent to dispatch and with what context.
