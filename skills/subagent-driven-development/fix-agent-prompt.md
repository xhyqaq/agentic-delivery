# Fix Agent Subagent Prompt Template

**LIFECYCLE NOTICE:**
- 无状态一次性 subagent
- 返回修复代码后关闭
- 不要保持活动状态

---

**Purpose:** Fix issues found during code review with severity-appropriate strategy

**When to dispatch:** At Stage 6 (Review Loop) of agentic-delivery workflow, after reviewer reports issues

---

## How to Dispatch

```
Task tool:
  description: "Fix review issues"
  prompt: |
    You are fixing issues found during code review.

    ## Input
    - Review report: {review_report}
    - Task description: {task_description}
    - Working directory: {working_directory}
    - Severity: {severity}

    ### Context (severity-dependent)

    {context_block}

    ## Your Job

    Based on severity, use different fix strategies:

    ### For Minor Issues (small style/naming/trivial fixes)
    1. Read relevant files from checkpoint commit
    2. Apply surgical patches - minimal changes only
    3. Preserve existing structure and approach
    4. Example fixes:
       - Extract magic numbers to constants
       - Fix variable naming
       - Add missing error messages
       - Remove unused imports

    ### For Important Issues (code smells/maintainability)
    1. Read relevant files and understand context
    2. Make targeted refactoring - fix the issue properly
    3. May require modest restructuring
    4. Example fixes:
       - Extract duplicated code to helper functions
       - Simplify complex conditionals
       - Add proper error handling
       - Fix architectural violations

    ### For Critical Issues (wrong approach/security/data loss risk)
    1. **DO NOT look at original code** (to avoid bias)
    2. Read design spec and requirements carefully
    3. Reimplement from scratch based on spec
    4. Use correct patterns and architecture
    5. Example fixes:
       - Rewrite with proper security (SQL injection fix)
       - Use correct algorithm (wrong business logic)
       - Fix data race conditions
       - Implement missing core requirements

    ## Process

    1. **Analyze review report**
       - List all issues by severity
       - Group related issues
       - Identify dependencies

    2. **Ask if unclear** (before coding)
       - "The review says X is missing, but I'm not clear on the expected behavior"
       - "Should I refactor the entire module or just fix the flagged function?"

    3. **Implement fixes**
       - For Minor/Important: Patch existing code surgically
       - For Critical: Reimplement from spec (ignore original)
       - Test your changes locally
       - Verify all flagged issues are addressed

    4. **Create fix commit**
       ```bash
       git add [files]
       git commit -m "$(cat <<'EOF'
       fix: [brief description of what was fixed]

       - Issue 1: [what was wrong, what you fixed]
       - Issue 2: [what was wrong, what you fixed]

       Addresses review issues: [list issue IDs if available]
       EOF
       )"
       ```

       **IMPORTANT:**
       - Always use HEREDOC format (`<<'EOF'`) to avoid shell escaping issues
       - Verify fix commit was created: `git log -1 --oneline`
       - Report the fix commit SHA in your output

    5. **Self-verify**
       - Run tests (if any)
       - Check that all issues from review are addressed
       - List any concerns or edge cases

    6. **Report back** (see Output Format below)

    ## Status Rules

    - **DONE**: All issues fixed successfully, confident in solution
    - **BLOCKED**: Cannot fix without architectural decision or user clarification
    - **NEEDS_CONTEXT**: Missing information to complete fix

    ## Output Format

    Return JSON:
    {
      "status": "DONE" | "BLOCKED" | "NEEDS_CONTEXT",
      "fix_commit": "abc123def",
      "changes": [
        {
          "file": "src/auth/jwt.ts",
          "description": "Extracted TOKEN_EXPIRY constant, fixed magic number"
        },
        {
          "file": "src/auth/validation.ts",
          "description": "Added proper error handling for invalid tokens"
        }
      ],
      "issues_addressed": [1, 2, 3],
      "verification": {
        "tests_passed": true,
        "linter_clean": true,
        "self_review": "All flagged issues resolved"
      },
      "concerns": []
    }

    If status is BLOCKED:
    {
      "status": "BLOCKED",
      "blocker": "Cannot determine correct error code without clarifying business rule",
      "questions": [
        "Should invalid tokens return 401 or 403?",
        "Do we want to log failed attempts?"
      ]
    }

  subagent_type: "general-purpose"
```

---

## Input Parameters (from orchestrator)

| Parameter | Type | Example | Required |
|-----------|------|---------|----------|
| review_report | JSON/object | Full review output with issues | ✅ |
| task_description | string | "Implement JWT authentication" | ✅ |
| working_directory | string | "/Users/xhy/Desktop/my-project" | ✅ |
| severity | string | "Minor" \| "Important" \| "Critical" | ✅ |

### For Minor/Important Issues (additional context)

| Parameter | Type | Example | Required |
|-----------|------|---------|----------|
| relevant_files | array | ["src/auth/jwt.ts", "src/auth/validation.ts"] | ✅ |
| checkpoint_commit | string | "abc123def" | ✅ |

### For Critical Issues (additional context)

| Parameter | Type | Example | Required |
|-----------|------|---------|----------|
| design_spec | string | Complete requirements from design-spec.md | ✅ |
| **NO original code** | - | To avoid bias toward flawed approach | ✅ |

---

## Context Block Templates

### For Minor/Important (patch strategy)

```markdown
### Relevant Files (from checkpoint {checkpoint_commit})
{file_contents}

### Original Implementation Context
- The code works but has quality issues
- Your goal: surgical fixes, preserve structure
- Don't over-engineer or add features
```

### For Critical (reimplement strategy)

```markdown
### Design Spec Requirements
{design_spec_excerpt}

### What Went Wrong
{review_report_summary}

### Your Approach
- **DO NOT read the original code**
- Implement from spec as if starting fresh
- Use correct patterns from the start
- Ignore the flawed approach entirely
```

---

## Output Contract

**Success (DONE):**
```json
{
  "status": "DONE",
  "fix_commit": "sha",
  "changes": [{file, description}],
  "issues_addressed": [1, 2, 3],
  "verification": {...},
  "concerns": []
}
```

**Blocked (BLOCKED):**
```json
{
  "status": "BLOCKED",
  "blocker": "description",
  "questions": ["Q1", "Q2"]
}
```

**Need Context (NEEDS_CONTEXT):**
```json
{
  "status": "NEEDS_CONTEXT",
  "missing": ["what", "why"],
  "tried": "what you attempted"
}
```

---

## Quality Gates

- All issues from review report are addressed (or explained why not)
- Fix commit created with clear message
- Self-verification completed (tests, linter)
- For Critical: reimplementation follows spec (no original code influence)
- For Minor/Important: changes are surgical and minimal

---

## See Also

- **Fix strategy decision logic:** `review-fix-strategy/SKILL.md`
- **When to use each severity:** `review-fix-strategy/SKILL.md` § Part 1
- **Integration with review loop:** `subagent-driven-development/SKILL.md` § Stage 6
