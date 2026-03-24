---
name: agentic-delivery
description: "Use when user requests new features, enhancements, bug fixes, or any development task that involves code changes — handles intent recognition, requirement analysis, design, implementation, code review, and documentation"
---

# Agentic Delivery

End-to-end delivery orchestration. Main agent orchestrates, subagents execute.

**Announce at start:** "I'm using the agentic-delivery skill to handle this task."

**CRITICAL:** Read `runtime-policies.md` for enforcement rules. These are HARD RULES that cannot be bypassed.

## The Rule

**Identify intent FIRST, then route to the correct path.** Do not start coding before completing the appropriate upstream stages.

## Stage 0: Intent Recognition

Analyze the user's request and classify:

| Type | Signal | Path |
|------|--------|------|
| Large Feature | Multi-module, multi-file, needs design, architecture impact | Full Pipeline (Stage 1-7) |
| Small Change | Single file/function, clear scope | Fast Path (Stage 5-7 only) |
| Bug Fix | Explicit error behavior, needs root cause analysis | Debug Path |

**Cross-session reuse:** Check `docs/<project>/<feature>/` for existing artifacts. If `design-spec.md` exists → skip Stage 3. If `implementation-plan.md` exists → skip Stage 4.

## Full Pipeline (Large Feature)

### Stage 1: Doc Scan

Dispatch `doc-scanner` subagent to generate project context.

**Input:** Project root path, scan targets, doc template.
**Output:** `docs/<project>/project-context.md`
**Lifecycle:** Dispose after completion.

### Stage 2: Requirement Analysis & Design

Execute brainstorming process (main agent, NOT subagent):

1. Read `project-context.md`
2. Ask ONE question at a time to clarify requirements
3. Propose 2-3 approaches with trade-offs, give recommendation
4. Present design in sections, get user approval per section
5. Write to `docs/<project>/<feature>/design-spec.md`
6. Dispatch Spec Reviewer subagent → review loop (max 3 rounds)
7. User reviews written spec before proceeding

**REQUIRED SUB-SKILL:** Follow brainstorming skill.

### Stage 3: Implementation Plan

Main agent writes detailed plan:

1. Map file structure: which files to create/modify, exact paths
2. Decompose into bite-sized tasks (2-5 min each)
3. Each task: file paths, steps, test commands, expected results, commit message
4. Mark task dependencies (parallel vs sequential)
5. Dispatch Plan Reviewer subagent → review loop (max 3 rounds)

**Parallel constraint (HARD RULE):**
> Tasks sharing ANY file MUST be sequential. Only tasks with zero file overlap may be parallel.
> This is decided at plan time, NOT execution time.

**Output:** `docs/<project>/<feature>/implementation-plan.md`

**REQUIRED SUB-SKILL:** Follow writing-plans skill.

### Stage 4: Code Implementation

**YOU MUST dispatch one Implementer subagent per task. DO NOT implement tasks inline.**

**Enforcement rules:**
1. When the plan contains distinct frontend and backend slices, you MUST dispatch dedicated subagents with explicit ownership
2. When tasks are marked as independent/parallel, you MUST dispatch them concurrently
3. You MUST NOT write code yourself — only coordinate subagents
4. If subagent dispatch is unavailable (tool error, platform limitation), STOP and report the blocker to the user. DO NOT silently degrade to single-agent mode.

**Pre-execution validation:**
Before entering Stage 4, verify:
- Subagent dispatch capability is available (spawn_agent or Task tool)
- `implementation-plan.md` exists and contains tasks
- You have read and parsed all tasks from the plan

**Orchestrator responsibilities:**
1. Extract ALL tasks with full text from plan (do NOT make subagent read plan file)
2. Route by dependency: independent tasks → parallel, dependent tasks → sequential
3. Construct minimum-context prompt per subagent (see Minimum Context §below)

**Implementer receives:**
- Task full text (copy-pasted, not file path)
- Relevant spec fragment only (not entire design doc)
- Working directory
- Interface definitions from prerequisite tasks (if dependent)

**Status handling:**

| Status | Action |
|--------|--------|
| DONE | Proceed to Stage 5 review |
| DONE_WITH_CONCERNS | Evaluate concerns, then review or address first |
| NEEDS_CONTEXT | Provide missing context, re-dispatch |
| BLOCKED | Assess: context gap → supplement; too complex → split task; plan wrong → back to Stage 3 |

**REQUIRED SUB-SKILL:** Use subagent-driven-development skill for dispatch pattern.

**ENFORCEMENT:** See `runtime-policies.md` § Required Subagents for failure handling and violation detection.

### Stage 5: Review Loop

Per-task, in strict order:

```
Implementer returns DONE
       │
       ▼
  Spec Compliance Review (subagent)
       │
       ├── Pass → Code Quality Review (subagent)
       │              │
       │              ├── Pass → Commit → Doc Sync → next task
       │              └── Fail → Fix (see review-fix-strategy) → re-review
       │
       └── Fail → Fix (see review-fix-strategy) → re-review
```

**Order is mandatory:** Spec Compliance FIRST, then Code Quality. No point reviewing quality if it doesn't meet spec.

**Max rounds:** 3 per review stage. Exceeds → escalate to user.

**Commit strategy:**
- Commit immediately after all reviews pass for each task
- Fix commits are separate from implementation commits

**After commit:** Dispatch `doc-syncer` subagent to update `docs/<project>/<feature>/changelog.md`.

**REQUIRED SUB-SKILL:** Use review-fix-strategy for fix decisions.

### Stage 6: Summary

Main agent compiles delivery report:

- Requirement type and total task count
- Which tasks ran in parallel
- Review rounds and fix count
- All artifacts produced (docs, files changed)
- File change list with descriptions
- **Token Usage Report** (see token-monitor.md):
  - Estimated token consumption by stage (Spec, Plan, Implementer, Reviewers)
  - Total input tokens and cost savings from optimizations
  - Optimization opportunities identified

## Fast Path (Small Change)

```
Intent → Small Change
  → Stage 4: Implement (main agent can do directly, subagent optional)
  → Stage 5: Code Quality Review only (skip Spec Compliance)
  → Commit → Brief summary
```

## Debug Path (Bug Fix)

```
Intent → Bug Fix
  → Systematic Debugging (main agent, 4-phase)
  → Stage 5: Code Quality Review only
  → Commit → Summary with root cause explanation
```

**REQUIRED SUB-SKILL:** Use systematic-debugging skill.

## Minimum Context Principle

> Give each subagent ONLY what it needs to complete its task. Never pass full session history.

| Subagent | Receives | Does NOT receive |
|----------|----------|-----------------|
| Doc Scanner | Project root, scan targets, template | User requirements, design docs |
| Spec Reviewer | Design doc path, review checklist | Project code, plan |
| Plan Reviewer | Plan path, design doc path | Code details |
| Implementer (backend) | Task text, backend conventions, API schema | Frontend code/conventions, other tasks |
| Implementer (frontend) | Task text, frontend conventions, API schema | Backend code/conventions, other tasks |
| Spec Compliance Reviewer | Task requirements, implementer report | Other tasks, global context |
| Code Quality Reviewer | Git diff (BASE_SHA..HEAD_SHA), task description | Requirements doc, other tasks |
| Fix Agent | Review report + code (Minor) or spec only (Critical) | Other tasks, global context |
| Doc Syncer | Task description, changed files, changelog template | Code details, review process |

**Anti-patterns:**
- Do NOT let subagent read plan file — paste task text into prompt
- Do NOT pass session history — construct precise prompt
- Do NOT give all docs to one subagent — filter by role

## Model Selection

Choose the **cheapest model that can handle the role**:

| Role | Model Level | Reason |
|------|------------|--------|
| Doc Scanner | Fast | Structured extraction, no deep reasoning |
| Spec/Plan Reviewer | Capable | Needs completeness and consistency judgment |
| Implementer (simple) | Fast | 1-2 files, clear spec, mechanical |
| Implementer (complex) | Standard | Multi-file coordination |
| Code Quality Reviewer | Capable | Needs architecture judgment |
| Fix Agent | Same or higher than original task | Fix needs at least equal understanding |
| Doc Syncer | Fast | Template filling, mechanical |

**Complexity signals:**
- 1-2 files + complete spec → Fast model
- Multi-file + integration → Standard model
- Design judgment + broad understanding → Capable model

## Platform Adaptation

Skills use Claude Code tool names. See `references/codex-tools.md` for Codex equivalents.

## Red Flags

**Never:**
- Start coding before completing upstream stages (for large features)
- **Implement tasks inline when executing Stage 4 (MUST use subagents)**
- **Silently fall back to single-agent mode without reporting blockers**
- Skip review stages
- Parallelize tasks that share files
- Pass full context to subagents
- Let subagent read plan file directly
- Proceed with Critical review issues unfixed

**Always:**
- Identify intent before acting
- Check for existing docs (cross-session reuse)
- Commit after each task's review passes
- Escalate to user after 3 failed review rounds
