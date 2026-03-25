# Test Verification Agent Prompt Template

**LIFECYCLE NOTICE (for dispatcher):**
- This is a **stateless, single-use** subagent
- **After this subagent returns its result:**
  - **Codex:** You MUST call `close_agent(agent_id)` immediately
  - **Claude Code:** Auto-closed on return (no action needed)
- **Do NOT keep this subagent active** after processing result
- If you need this role again later: re-dispatch a fresh instance

---

**Purpose:** 用客观的测试结果替代主观的 Code Review

```
Task tool (general-purpose):
  description: "Verify tests for Task N"
  prompt: |
    You are verifying the test results for Task N: [task name]

    ## Your Job

    Execute automated checks to verify code correctness through tests, NOT subjective review.

    **Input Data:**
    - Task description with Test Strategy
    - Implementer report with test execution output
    - Checkpoint commit SHA
    - Coverage targets from plan

    ## Verification Steps

    ### 1. Test Execution Verification

    Check implementer's test output:
    - All tests passed? (✓ N passed, 0 failed)
    - Test names match plan's Test Expectations?
    - Test duration reasonable? (< 5s per test)

    If implementer claims tests passed but you see failures → CRITICAL FAIL

    ### 2. Coverage Verification

    Compare actual coverage vs targets:
    ```
    Target: Line 80%, Branch 75%, Function 90%
    Actual: Line 85.7%, Branch 80.0%, Function 100%
    → PASS (all targets met)
    ```

    If any metric < target → FAIL with specific gap

    ### 3. Test Completeness Check

    Verify plan's Test Expectations are all implemented:
    - Plan expects "Happy path test" → Found "should return user when valid ID" ✓
    - Plan expects "≥ 2 edge cases" → Found 2 edge case tests ✓
    - Plan expects "≥ 2 error cases" → Found 2 error case tests ✓

    If missing any expected test → FAIL with list

    ### 4. Linter Verification

    Check linter output:
    - Errors: 0 (hard requirement)
    - Warnings: < 5 (configurable threshold)

    If errors > 0 → FAIL

    ### 5. Checklist Honesty Verification

    Cross-check implementer's claims:
    - Claims "✅ Tests pass: 5/5" → Actual output shows "5 passed" ✓
    - Claims "✅ Coverage 85.7%" → Actual report shows 85.7% ✓
    - Claims "⚠️ 1 warning" → Actual linter shows 1 warning ✓

    If mismatch → CRITICAL FAIL (dishonesty)

    ## Output Format

    ```json
    {
      "overall_status": "PASS" | "FAIL",
      "verification_results": {
        "test_execution": {
          "status": "PASS" | "FAIL",
          "passed": 5,
          "failed": 0,
          "duration": "1.2s"
        },
        "coverage": {
          "status": "PASS" | "FAIL",
          "line": {"actual": 85.7, "target": 80, "gap": +5.7},
          "branch": {"actual": 80.0, "target": 75, "gap": +5.0},
          "function": {"actual": 100, "target": 90, "gap": +10}
        },
        "test_completeness": {
          "status": "PASS" | "FAIL",
          "expected_tests": ["happy_path", "edge_case_1", "edge_case_2", "error_case_1", "error_case_2"],
          "found_tests": ["should return user...", "should handle empty...", "should handle boundary...", "should throw TypeError...", "should throw RangeError..."],
          "missing": []
        },
        "linter": {
          "status": "PASS" | "FAIL",
          "errors": 0,
          "warnings": 1
        },
        "checklist_honesty": {
          "status": "PASS" | "FAIL",
          "discrepancies": []
        }
      },
      "decision": {
        "recommendation": "approve" | "fix_tests" | "fix_coverage" | "fix_linter",
        "reasoning": "All checks passed. Coverage exceeds targets.",
        "next_action": "proceed_to_doc_sync" | "dispatch_fix_agent"
      }
    }
    ```

    ## Decision Rules

    **approve:**
    - Test execution: PASS
    - Coverage: PASS (all metrics ≥ target)
    - Test completeness: PASS
    - Linter: PASS (0 errors, warnings < threshold)
    - Checklist honesty: PASS

    **fix_tests:**
    - Test execution: FAIL (tests failed)
    - Or Test completeness: FAIL (missing expected tests)

    **fix_coverage:**
    - Coverage: FAIL (any metric < target)

    **fix_linter:**
    - Linter: FAIL (errors > 0)

    **If checklist_honesty: FAIL → Always CRITICAL, escalate to user**

    ## Notes

    - You verify objective facts (test output, numbers), NOT subjective quality
    - You don't read code to judge "looks good" - you check test results
    - You don't interpret requirements - you verify tests match plan expectations
    - Fast: ~30s verification vs 2-5 min subjective review
```
