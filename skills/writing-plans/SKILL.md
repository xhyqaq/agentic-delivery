---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/<project>/<feature>/implementation-plan.md`
- Default format: `docs/<project-name>/<feature-name>/implementation-plan.md`
- Automatically detect project name from git root or current directory
- User preferences (from project CLAUDE.md or ~/.claude/config) override this default
- Legacy path `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md` still supported for compatibility

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

When marking tasks as parallel, also check for **indirect overlaps**: lock files, barrel exports, auto-generated files. These can cause merge conflicts even when source files don't overlap.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Create a checkpoint commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development skill (recommended) or executing-plans skill *(external, not included)* to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interface Contract:**
```python
def function(input: InputType) -> OutputType:
    """
    Purpose: What this function does and why

    Args:
        input: Description of input expectations

    Returns:
        Description of output format

    Raises:
        SpecificError: When and why
    """
```

**Test Expectations:**
```python
# Test case 1: Normal case
function(valid_input) → expected_output

# Test case 2: Edge case
function(edge_input) → edge_output

# Test case 3: Error case
function(invalid_input) → raises SpecificError
```

**Steps:**

- [ ] **Step 1: Write failing tests based on contract**
  Run: `pytest tests/path/test.py::test_name -v`
  Expected: FAIL with "function not defined"

- [ ] **Step 2: Implement function to satisfy contract**
  Implementation approach: [High-level strategy, e.g., "Use caching decorator for performance"]

- [ ] **Step 3: Verify tests pass**
  Run: `pytest tests/path/test.py::test_name -v`
  Expected: PASS (all 3 test cases)

- [ ] **Step 4: Create checkpoint commit**
  ```bash
  git add tests/path/test.py src/path/file.py
  git commit -m "feat: add specific feature"
  ```
````

## Remember
- Exact file paths always
- **Interface contracts and test expectations** (NOT complete implementations)
  - Define what functions should do (signature + behavior)
  - Show test expectations (input → expected output)
  - Let implementer figure out HOW to achieve it
- **Provide code ONLY when**:
  - Demonstrating a specific pattern to follow
  - Showing exact API usage for unfamiliar libraries
  - Defining type interfaces or data structures
- Exact commands with expected output
- In execution plans, "commit" means a review checkpoint commit unless explicitly stated otherwise
- Task checkboxes (`- [ ]`) are used for cross-session progress tracking. After a task completes both review stages, the orchestrator adds a test summary section before marking steps as complete:
  ```markdown
  ### Task N: [Component Name]

  **Tests:** (added after completion)
  - ✓ [test case 1]
  - ✓ [test case 2]
  **Test file:** `path/to/test.file`
  **Commit:** [SHA]

  **Files:**
  ...
  ```
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits

## Plan Review Loop

After writing the complete plan:

1. Dispatch a single plan-document-reviewer subagent (see plan-document-reviewer-prompt.md) with precisely crafted review context — never your session history. This keeps the reviewer focused on the plan, not your thought process.
   - Provide: path to the plan document, path to spec document
2. If ❌ Issues Found: fix the issues and present the updated plan to the user for approval
3. If ✅ Approved: proceed to execution handoff

**Review loop guidance:**
- Plan review is **1 round only** — no re-dispatch cycle
- If the reviewer finds issues, fix them and let the user decide whether to proceed
- Plans are validated again during implementation; over-reviewing at this stage wastes tokens and subagent slots

## Execution Handoff

After saving the plan, automatically proceed to subagent-driven development:

**"Plan complete and saved to `docs/<project>/<feature>/implementation-plan.md`. Starting subagent-driven execution..."**

**REQUIRED SUB-SKILL:** Invoke subagent-driven-development skill to execute the plan.
- Fresh subagent per task
- Two-stage review (spec compliance → code quality)
- Fast iteration with isolated contexts
