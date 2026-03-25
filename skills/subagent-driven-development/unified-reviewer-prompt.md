# Unified Reviewer Prompt Template

**LIFECYCLE NOTICE (for dispatcher):**
- This is a **stateless, single-use** subagent
- **After this subagent returns its result:**
  - **Codex:** You MUST call `close_agent(agent_id)` immediately
  - **Claude Code:** Auto-closed on return (no action needed)
- **Do NOT keep this subagent active** after processing result
- If you need this role again later: re-dispatch a fresh instance

---

**Purpose:** Combined spec compliance + code quality review in a single pass (2x faster than sequential)

**Why unified:** Both reviewers read the same git diff. Combining them reduces subagent overhead and token usage by ~40%.

**Critical:** This reviewer performs TWO SEQUENTIAL checks (spec first, then quality). Spec failures block quality review.

```
Task tool (general-purpose):
  description: "Unified review for Task N"
  prompt: |
    You are performing a two-stage review of an implementation.

    **STAGE 1: Spec Compliance (MUST check first)**
    **STAGE 2: Code Quality (ONLY if Stage 1 passes)**

    ## What Was Requested

    [FULL TEXT of task requirements]

    ## What Implementer Claims They Built

    [From implementer's report, including their Pre-Submission Checklist]

    ## Code Changes

    [GIT DIFF of the checkpoint commit — use `git diff BASE_SHA..HEAD_SHA` to generate]

    ## Project Context

    [Project conventions, patterns, tech stack]

    ---

    ## STAGE 1: Spec Compliance Check (MANDATORY FIRST)

    **CRITICAL: Do Not Trust the Report or Checklist**

    The implementer may have missed things or been overly optimistic. You MUST verify independently.

    **DO NOT:**
    - Take their word for what they implemented
    - Trust their Pre-Submission Checklist without verification
    - Accept their interpretation of requirements

    **DO:**
    - Read the actual code they wrote
    - Compare actual implementation to requirements line by line
    - Check for missing pieces they claimed to implement
    - Look for extra features they didn't mention

    ### Verify:

    **Missing requirements:**
    - Did they implement everything that was requested?
    - Are there requirements they skipped or missed?
    - Did they claim something works but didn't actually implement it?
    - Check each requirement from spec against actual code

    **Extra/unneeded work:**
    - Did they build things that weren't requested?
    - Did they over-engineer or add unnecessary features?
    - Did they add "nice to haves" that weren't in spec?

    **Misunderstandings:**
    - Did they interpret requirements differently than intended?
    - Did they solve the wrong problem?
    - Did they implement the right feature but wrong way?

    **API Contract Compliance (if applicable):**
    - Field names match spec exactly (camelCase vs snake_case)
    - Types match (string vs number vs boolean)
    - Response structures match
    - Status codes match

    **If ANY spec issues found → STOP. Do not proceed to Stage 2.**
    **If all spec requirements met → Proceed to Stage 2.**

    ---

    ## STAGE 2: Code Quality Check (ONLY if Stage 1 passed)

    **Only perform this stage if Stage 1 found ZERO spec issues.**

    Evaluate implementation quality using three severity tiers:

    ### Issue Severity Classification

    **Critical (MUST fix before approval):**
    - Security vulnerabilities (injection, XSS, auth bypass)
    - Data loss or corruption risks
    - Production-breaking bugs (crashes, infinite loops)
    - Fundamentally wrong architecture (requires rewrite)
    - Missing error handling for failure-critical operations

    **Important (should fix, may require re-review):**
    - Significant code smells (duplicated logic, God objects)
    - Hard-to-maintain patterns (magic numbers, unclear naming)
    - Missing tests for edge cases
    - Performance issues (N+1 queries, memory leaks)
    - Inconsistent with project conventions

    **Minor (nice to fix, can merge as-is):**
    - Nitpicks (formatting, minor naming improvements)
    - Missing comments for complex logic
    - Could be more elegant
    - Trivial optimizations

    ### Verification Against Implementer's Checklist

    Cross-check their Pre-Submission Checklist claims:
    - Did they report "✅ No magic numbers" but you found some? → Flag as missed
    - Did they report "⚠️ Found 1 TODO" and you confirm it? → Validate their honesty
    - Did they claim "✅ All tests pass" but you see test gaps? → Important issue

    **If implementer's checklist was accurate → Trust +1 (note in Strengths)**
    **If implementer missed obvious issues → Flag as concern**

    ### Quality Dimensions to Check

    **Code Organization:**
    - Each file has one clear responsibility?
    - Well-defined interfaces?
    - Following file structure from plan?
    - New/modified files appropriately sized?

    **Code Cleanliness:**
    - No magic numbers (should be named constants)
    - No debug code (console.log, commented blocks)
    - No TODO/FIXME left unresolved
    - Naming clear and consistent with project
    - Error handling complete

    **Testing:**
    - Tests cover core paths (not just happy path)
    - Edge cases tested (as per spec)
    - Tests actually pass
    - Test names describe what they verify

    **Maintainability:**
    - Code is understandable
    - No over-engineering (YAGNI)
    - Follows existing patterns in codebase
    - Dependencies reasonable

    ---

    ## Output Format (REQUIRED STRUCTURE)

    ```json
    {
      "stage_1_spec_compliance": {
        "status": "pass" | "fail",
        "verification_notes": "Cross-checked N requirements against actual code...",
        "missing_requirements": [
          "Requirement X not implemented (claimed in checklist but missing in code)"
        ],
        "extra_implementation": [
          "Added feature Y that wasn't in spec (file.ts:123)"
        ],
        "misunderstandings": [
          "Interpreted requirement Z as ... but spec meant ..."
        ],
        "implementer_checklist_accuracy": "accurate" | "optimistic" | "missed_issues"
      },
      "stage_2_code_quality": {
        "status": "pass" | "fail" | "skipped",
        "skipped_reason": "Stage 1 failed" | null,
        "strengths": [
          "Excellent test coverage (8 tests for all edge cases)",
          "Clean separation of concerns",
          "Implementer's self-review was thorough and accurate"
        ],
        "issues": [
          {
            "severity": "Critical" | "Important" | "Minor",
            "description": "Magic number 3600 at auth.ts:45 (should be TOKEN_EXPIRY constant)",
            "location": "auth.ts:45"
          }
        ],
        "overall_quality": "excellent" | "good" | "needs_work"
      },
      "final_decision": {
        "recommendation": "approve" | "fix_spec_first" | "fix_quality" | "fix_both",
        "reasoning": "...",
        "estimated_fix_effort": "trivial" | "minor" | "moderate" | "major"
      }
    }
    ```

    ## Decision Rules

    **approve:**
    - Stage 1 = pass (or only trivial spec deviations agreed with implementer's concerns)
    - Stage 2 = pass (no Critical/Important issues, or only Minor nitpicks)

    **fix_spec_first:**
    - Stage 1 = fail (missing requirements or extra features)
    - Stage 2 = skipped (don't review quality until spec is right)

    **fix_quality:**
    - Stage 1 = pass
    - Stage 2 = fail (Critical or Important issues found)

    **fix_both:**
    - Stage 1 = fail
    - Stage 2 = fail (rare: both spec and quality problems)

    ## Why This Order Matters

    **Spec compliance MUST be checked first because:**
    1. No point reviewing code quality if it's the wrong code
    2. Fix strategy differs: spec issues may require reimplementation
    3. Quality issues can be patched; spec issues need rework

    **Example:**
    - If implementer missed a requirement → They need to add it (spec issue)
    - If implementer has magic numbers → They need to extract constants (quality issue)
    - Fixing spec issues first prevents wasted work on quality of wrong code

    ## Notes

    - Be precise with file:line references for all issues
    - Distinguish between "implementer claimed X but didn't do it" vs "implementer did X but not in spec"
    - If implementer's Pre-Submission Checklist was accurate, praise it (reinforces good behavior)
    - If implementer missed issues they should have caught, note it (helps improve future self-reviews)
```
