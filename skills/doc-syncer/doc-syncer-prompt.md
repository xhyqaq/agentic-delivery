# Doc Syncer Subagent Prompt Template

**LIFECYCLE NOTICE:**
- 无状态一次性 subagent
- 更新 changelog 后关闭
- 不要保持活动状态

---

**Purpose:** Update project changelog after a task's review loop passes

**When to dispatch:** At Stage 6 (After Reviews Pass) of agentic-delivery workflow, after each task completes review

---

## How to Dispatch

```
Task tool:
  description: "Update changelog"
  prompt: |
    You are updating the project changelog after a task has passed review.

    ## Input
    - Task name: {task_name}
    - Task description: {task_description}
    - Changed files: {changed_files}
    - Review summary: {review_summary}
    - Changelog path: {changelog_path}
    - Today's date: {date}

    ## Your Job

    Follow the workflow in doc-syncer/SKILL.md:

    1. **Read existing changelog** (if exists)
       - Use Read tool to get current content
       - Parse markdown structure
       - Identify where to append

    2. **Append new entry using template exactly:**
       ```markdown
       ### Task N: [Task Name]
       - **Date**: YYYY-MM-DD
       - **Changed Files**:
         - `path/to/file.ts` — Created / Modified — [one-line description]
         - `path/to/test.ts` — Created — [one-line description]
       - **Change Summary**: [2-3 sentences describing what changed and why]
       - **Review**: Passed (N rounds, fixes: [brief description or "none"])
       ```

    3. **Write updated changelog** back to file
       - Use Write tool
       - Preserve all existing entries
       - Append new entry at the end

    ## Important Rules

    - Do NOT rewrite existing changelog entries — append only
    - Do NOT add commentary or opinions — factual record only
    - Do NOT change section headings of the template
    - Do NOT include code snippets in the changelog
    - Do NOT read or analyze the actual code changes — use the provided summary

    See doc-syncer/SKILL.md for complete details.

    ## Output Format

    Return JSON:
    {
      "status": "DONE",
      "changelog_path": "docs/<project>/<feature>/changelog.md",
      "entry_added": true,
      "entry_preview": "### Task 3: User auth API\n- **Date**: 2026-03-25\n..."
    }

  subagent_type: "general-purpose"
```

---

## Input Parameters (from orchestrator)

| Parameter | Type | Example |
|-----------|------|---------|
| task_name | string | "Task 3: User authentication API" |
| task_description | string | "Implement REST API for user authentication with JWT tokens" |
| changed_files | array | `[{"path": "src/auth/jwt.ts", "status": "Created", "description": "JWT token generation and validation"}, ...]` |
| review_summary | string | "Passed (2 rounds, fixes: token expiry validation)" |
| changelog_path | string | "docs/my-project/auth-feature/changelog.md" |
| date | string | "2026-03-25" |

---

## Output Contract

**Success (DONE):**
```json
{
  "status": "DONE",
  "changelog_path": "docs/<project>/<feature>/changelog.md",
  "entry_added": true
}
```

**Failure scenarios:**
- Changelog file cannot be written → return error
- Template malformed → return error

---

## Quality Gates

- Changelog entry follows template exactly
- All required fields present (Date, Changed Files, Change Summary, Review)
- No rewrites of existing entries (append only)
- Factual record only (no opinions)
- One-line file descriptions are clear and concise

---

## See Also

- **Full workflow details:** `doc-syncer/SKILL.md`
- **Changelog entry template:** `doc-syncer/SKILL.md` § Changelog Entry Template
- **What NOT to do:** `doc-syncer/SKILL.md` § What NOT to Do
