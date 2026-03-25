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

## Authority and Conflict Resolution

`agentic-delivery` is the orchestration shell for this workflow. It decides routing, stage transitions, and the input/output contract for each stage. It does **not** redefine the internal methods of downstream stage-owner skills.

**Priority order when instructions conflict:**
1. `runtime-policies.md`
2. Stage-owner skill for the current stage
3. `agentic-delivery`
4. Prompt templates
5. Explanatory docs (`docs/design`, `README.md`, examples)

**Stage owners:**
- Stage 3 (Requirement Analysis & Design) → `brainstorming`
- Stage 4 (Implementation Plan) → `writing-plans`
- Stage 5-6 (Implementation + Review Loop) → `subagent-driven-development`
- Review fix routing → `review-fix-strategy`
- Bug path → `systematic-debugging`

## Git Branching Strategy

**Full Pipeline:**
- After Stage 1 intent classification, create a feature branch: `feature/<feature-name>`
- All implementation and doc changes happen on this branch
- Stage 7 summary is the last action on the branch
- Branch integration (merge, PR, or cleanup) is handled by `finishing-a-development-branch` skill

**Fast Path / Debug Path:**
- For single-file changes: work on current branch (typically already a feature branch)
- If on main/master: create a branch first — never commit directly to main

**Parallel implementers:** All work on the same feature branch. Since parallel tasks have zero file overlap (enforced at plan time), their commits will not conflict.

## Stage 1: Intent Recognition

Analyze the user's request and classify:

| Type | Signal | Path |
|------|--------|------|
| Large Feature | Multi-module, multi-file, needs design, architecture impact | Full Pipeline (Stage 1-7) |
| Small Change | Single file/function, clear scope | Fast Path (Stage 5-7 only) |
| Bug Fix | Explicit error behavior, needs root cause analysis | Debug Path |

**Cross-session reuse:** Check `docs/<project>/<feature>/` for existing artifacts:
- `design-spec.md` exists → skip Stage 3
- `implementation-plan.md` exists → skip Stage 4
- `implementation-plan.md` has tasks marked `[x]` → skip completed tasks in Stage 5, resume from first unchecked task

## Full Pipeline (Large Feature)

### Stage 2: Doc Scan

Dispatch `doc-scanner` subagent to generate project context.

**Strategy:** "Trust but Verify" - Read external docs as claims, verify critical assertions (tech stack, architecture) via code sampling, trust verified info while discarding contradictions.

**Process:**
1. Pre-check: Document freshness (skip if > 1 year old)
2. Read & Verify: Sample 10-20 files to validate claims
3. Fallback: Code-only scan if docs contradicted/missing
4. Generate: `project-context.md` with verification status (✅ Verified / ⚠️ Partial / ❌ Contradicted)

**Input:** Project root path
**Output:** `docs/<project>/project-context.md` (with verification markers)
**Cost:** ~300 tokens (verified) or ~2000 tokens (code-only fallback)
**Lifecycle:** Dispose after completion.

### Stage 3: Requirement Analysis & Design

Execute brainstorming process (main agent, NOT subagent):

1. Read `project-context.md`
2. Follow the `brainstorming` questioning strategy to clarify requirements
   - Batch independent questions when possible
   - Ask sequentially only when later questions depend on earlier answers
3. Propose 2-3 approaches with trade-offs, give recommendation
4. Present design in sections, get user approval per section
5. Write to `docs/<project>/<feature>/design-spec.md`
6. Dispatch Spec Reviewer subagent → review loop (max 3 rounds)
7. User reviews written spec before proceeding

**REQUIRED SUB-SKILL:** Follow brainstorming skill.
`agentic-delivery` routes into this stage; `brainstorming` owns the detailed questioning method.

### Stage 4: Implementation Plan

Main agent writes detailed plan:

1. Map file structure: which files to create/modify, exact paths
2. Decompose into bite-sized tasks (2-5 min each)
3. Each task: file paths, steps, test commands, expected results, commit message
4. Mark task dependencies (parallel vs sequential)
5. Dispatch Plan Reviewer subagent → if issues found, fix and escalate to user

**Parallel constraint (HARD RULE):**
> Tasks sharing ANY file MUST be sequential. Only tasks with zero file overlap may be parallel.
> This is decided at plan time, NOT execution time.

**Output:** `docs/<project>/<feature>/implementation-plan.md`

**REQUIRED SUB-SKILL:** Follow writing-plans skill.

### Stage 4 → Stage 5 Transition

**No user approval gate.** After plan review passes (or issues fixed), automatically proceed to Stage 5.

**Rationale:**
- Design (Stage 3) already user-approved → direction confirmed
- Plan review ensures quality → completeness, consistency verified
- Plan is mechanical breakdown → no strategic decisions
- User can interrupt anytime (live session)
- Checkpoint commits allow rollback (low risk)

**Announce transition:**
```
Plan review passed ✓
Starting Stage 5: Implementation
Dispatching N implementers for parallel execution...
```

**Exception:** If user explicitly requests "let me review the plan first" before Stage 4 completes, pause and wait. But this is **not default behavior**.

### Stage 5: Code Implementation

**YOU MUST dispatch one Implementer subagent per task. DO NOT implement tasks inline.**

**Enforcement rules:**
1. When the plan contains distinct frontend and backend slices, you MUST dispatch dedicated subagents with explicit ownership
2. When tasks are marked as independent/parallel, you MUST dispatch them concurrently
3. You MUST NOT write code yourself — only coordinate subagents
4. If subagent dispatch is unavailable (tool error, platform limitation), STOP and report the blocker to the user. DO NOT silently degrade to single-agent mode.

**Pre-execution validation:**
Before entering Stage 5, verify:
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
| DONE | Proceed to Stage 6 review |
| DONE_WITH_CONCERNS | Evaluate concerns, then review or address first |
| NEEDS_CONTEXT | Provide missing context, re-dispatch |
| BLOCKED | Assess: context gap → supplement; too complex → split task; plan wrong → back to Stage 4 |

**REQUIRED SUB-SKILL:** Use subagent-driven-development skill for dispatch pattern.

**After parallel batch completes:**
If multiple implementers ran in parallel, verify their commits merge cleanly:
1. Check for merge conflicts (lock files, barrel exports, auto-generated files)
2. Resolve mechanical conflicts (main agent, not subagent)
3. Run project test suite to verify integration
4. If non-trivial conflicts exist, the plan incorrectly classified tasks as parallel — fix plan or escalate

**ENFORCEMENT:** See `runtime-policies.md` § Required Subagents for failure handling and violation detection.

### Stage 6: Review Loop

Per-task, in strict order:

```
Implementer returns DONE
       │
       ▼
  Spec Compliance Review (subagent)
       │
       ├── Pass → Code Quality Review (subagent)
       │              │
       │              ├── Pass → Mark task complete → Doc Sync → next task
       │              └── Fail → Fix (see review-fix-strategy) → re-review
       │
       └── Fail → Fix (see review-fix-strategy) → re-review
```

**Order is mandatory:** Spec Compliance FIRST, then Code Quality. No point reviewing quality if it doesn't meet spec.

**Max rounds:** 3 per review stage. Exceeds → escalate to user.

**Commit strategy:**
- Implementer creates a checkpoint commit after implementation and local verification
- Each review-driven fix creates a separate fix commit
- A task is complete only after Spec Compliance and Code Quality both pass

**After reviews pass:** Dispatch `doc-syncer` subagent to update `docs/<project>/<feature>/changelog.md`.

**REQUIRED SUB-SKILL:** Use review-fix-strategy for fix decisions.

### Stage 7: Summary

Main agent compiles delivery report:

- Requirement type and total task count
- Which tasks ran in parallel
- Review rounds and fix count
- All artifacts produced (docs, files changed)
- File change list with descriptions
- Lessons learned or optimization opportunities

## Fast Path (Small Change)

```
Intent → Small Change
  → Stage 5: Implement (single task: main agent directly; 2+ tasks: dispatch subagents per runtime-policies)
  → Stage 6: Code Quality Review only (skip Spec Compliance)
  → Doc Sync (if changelog exists for this feature)
  → Stage 7: Brief summary
```

Fast Path is for **single-task** changes. If the change turns out to need 2+ tasks, escalate to Full Pipeline or dispatch subagents per runtime-policies.

## Debug Path (Bug Fix)

```
Intent → Bug Fix
  → Systematic Debugging (main agent, 4-phase)
  → Stage 6: Code Quality Review only
  → Doc Sync (if changelog exists for this feature)
  → Stage 7: Summary with root cause explanation
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
| Spec Compliance Reviewer | Task requirements, implementer report, git diff (checkpoint commit) | Other tasks, global context |
| Code Quality Reviewer | Git diff (BASE_SHA..HEAD_SHA), task description | Requirements doc, other tasks |
| Fix Agent | Review report + relevant files from the latest checkpoint (Minor) or spec only (Critical) | Other tasks, global context |
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
- **Implement tasks inline when executing Stage 5 (MUST use subagents)**
- **Silently fall back to single-agent mode without reporting blockers**
- Skip review stages
- Parallelize tasks that share files
- Pass full context to subagents
- Let subagent read plan file directly
- Proceed with Critical review issues unfixed

**Always:**
- Identify intent before acting
- Check for existing docs (cross-session reuse)
- Create checkpoint commits before review; only mark a task complete after reviews pass
- Escalate to user after 3 failed review rounds
