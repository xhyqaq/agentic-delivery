---
name: doc-syncer
description: "Use after a task's review loop passes to update project documentation with the changes made, keeping changelog and docs in sync with implementation"
---

# Doc Syncer

Update project documentation after a task completes review. Ensures every code change is reflected in the changelog.

## When to Use

- Called by `agentic-delivery` at Stage 5, after each task's review loop passes and commit is done
- Called after fix commits that introduce significant changes

## Input (Minimum Context)

The dispatcher MUST provide exactly these, nothing more:

| Field | Content |
|-------|---------|
| Task name | e.g. "Task 3: User auth API" |
| Task description | Brief summary of what was implemented |
| Changed files | List of files created/modified with one-line descriptions |
| Review summary | How many rounds, what was fixed |
| Changelog path | `docs/<project>/<feature>/changelog.md` |
| Changelog template | The full template text (see below) |
| Today's date | YYYY-MM-DD |

**Do NOT provide:** Full code, review details, other tasks, session history.

## The Process

1. **Read existing changelog** (if exists)
2. **Append new entry** following the template exactly
3. **Write updated changelog** back to file

## Changelog Entry Template

Each entry MUST follow this exact structure:

```markdown
### Task N: [Task Name]
- **Date**: YYYY-MM-DD
- **Changed Files**:
  - `path/to/file.ts` — Created / Modified — [one-line description]
  - `path/to/test.ts` — Created — [one-line description]
- **Change Summary**: [2-3 sentences describing what changed and why]
- **Review**: Passed (N rounds, fixes: [brief description or "none"])
```

## What NOT to Do

- Do NOT rewrite existing changelog entries — append only
- Do NOT add commentary or opinions — factual record only
- Do NOT change section headings of the template
- Do NOT include code snippets in the changelog
- Do NOT read or analyze the actual code changes — use the provided summary

## Lifecycle

**Created by:** `agentic-delivery` main agent after each task's review passes
**Disposed:** Immediately after changelog is updated
**Output persists:** As file on disk
