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
    | **Test Strategy (CRITICAL)** | **Every task has Test Strategy section with Coverage Targets, Test Framework, Test File** |
    | **Test Expectations (CRITICAL)** | **Every task has ≥ 1 Happy Path + ≥ 2 Edge Cases + ≥ 2 Error Cases** |

    ## Test Strategy Requirements (NEW - MANDATORY)

    **For each task, verify:**

    1. **Test Strategy section exists** with all required fields:
       - Test Type (Unit/Integration/E2E)
       - Coverage Targets (Line ≥ 80%, Branch ≥ 75%, Function ≥ 90%)
       - Test Framework specified
       - Test File path provided

    2. **Test Expectations are complete**:
       - ≥ 1 Happy Path test (normal case)
       - ≥ 2 Edge Cases tests (boundary values, empty input, etc.)
       - ≥ 2 Error Cases tests (TypeError, ValueError, etc.)
       - Tests use actual test code, not abstract descriptions

    3. **Quality Gates defined**:
       - Linter requirements (0 errors, warnings < 5)
       - Type Check requirements
       - Security requirements

    **If any task is missing Test Strategy → Flag as CRITICAL issue.**

    **Example of insufficient test expectations:**
    ```
    ❌ BAD:
    # Test case 1: Normal case
    # Test case 2: Edge case
    # Test case 3: Error case

    (Too abstract, no actual test code)
    ```

    **Example of good test expectations:**
    ```
    ✅ GOOD:
    # 1. Happy Path
    test('should return user when valid ID'):
        result = getUser(1)
        assert result.id == 1

    # 2. Edge Case 1: Empty database
    test('should return null when user not found'):
        result = getUser(999)
        assert result == null

    # 2. Edge Case 2: Boundary value
    test('should handle MAX_SAFE_INTEGER'):
        result = getUser(Number.MAX_SAFE_INTEGER)
        assert result == null

    # 3. Error Case 1: Invalid type
    test('should throw TypeError for string ID'):
        expect(() => getUser('abc')).toThrow(TypeError)

    # 3. Error Case 2: Negative ID
    test('should throw RangeError for negative ID'):
        expect(() => getUser(-1)).toThrow(RangeError)
    ```

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
