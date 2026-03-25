# Plan Document Reviewer Prompt Template

**LIFECYCLE NOTICE (for dispatcher):**
- This is a **stateless, single-use** subagent
- **After this subagent returns its result:**
  - **Codex:** You MUST call `close_agent(agent_id)` immediately
  - **Claude Code:** Auto-closed on return (no action needed)
- **Do NOT keep this subagent active** after processing result
- If you need this role again later: re-dispatch a fresh instance

---

Use this template when dispatching a plan document reviewer subagent.

**Purpose:** Verify the plan is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** The complete plan is written.

```
Task tool (general-purpose):
  description: "Review plan document"
  prompt: |
    You are a plan document reviewer. Verify this plan is complete and ready for implementation.

    **Plan to review:** [PLAN_FILE_PATH]
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Plan covers spec requirements, no major scope creep |
    | Task Decomposition | Tasks have clear boundaries, steps are actionable |
    | Buildability | Could an engineer follow this plan without getting stuck? |

    ## Calibration

    **Only flag issues that would cause real problems during implementation.**
    An implementer building the wrong thing or getting stuck is an issue.
    Minor wording, stylistic preferences, and "nice to have" suggestions are not.

    Approve unless there are serious gaps — missing requirements from the spec,
    contradictory steps, placeholder content, or tasks so vague they can't be acted on.

    **Code detail calibration:**
    - ✅ Plan provides: Interface signatures, test expectations, high-level approach
    - ❌ Plan should NOT: Include complete function implementations line-by-line
    - Exception: OK to show complete code when demonstrating unfamiliar pattern or library usage
    - If plan has full implementations everywhere, flag as "Over-specified - provides complete code instead of contracts"

    ## Output Format

    ## Plan Review

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters for implementation]

    **Recommendations (advisory, do not block approval):**
    - [suggestions for improvement]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
