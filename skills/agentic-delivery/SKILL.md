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

### Phase 0: Project Context Preparation (Blocking Prerequisite)

**Trigger:** Large feature path (full pipeline)

**Responsibility:** Ensure `docs/project-context.md` exists before design begins

**Flow (Simplified):**

1. **Check if document exists**
   ```python
   # Pseudo-code
   if file_exists("docs/project-context.md"):
       log("✅ Project context document exists, using it")
       # Proceed to Stage 3
   else:
       # Launch doc-scanner (blocking mode)
       log("⚠️ Project context document missing, starting scan...")
   ```

2. **Launch doc-scanner (blocking mode)** (if needed)

   **Display progress message:**
   ```
   Building project context...
   This is a one-time scan, subsequent sessions will reuse this document.

   Estimated time: 30-90 seconds
   ```

   **Dispatch doc-scanner subagent (synchronous wait):**
   - Use `doc-scanner-prompt.md` template
   - `subagent_type: "Explore"`
   - **Wait for subagent to complete** (not async)
   - Verify `docs/project-context.md` has been generated

3. **Blocking checkpoint**
   ```python
   # Pseudo-code
   assert file_exists("docs/project-context.md"), "Project context document missing"
   assert file_readable("docs/project-context.md"), "Project context document unreadable"
   assert file_size("docs/project-context.md") > 500, "Project context document too small (may be incomplete)"

   log("✅ Phase 0 complete: project context ready")
   ```

**Only after passing the checkpoint can you proceed to Stage 3.**

**Fast Path / Debug Path:** Skip Phase 0, proceed directly to implementation.

**Why not check freshness?**
- Stage 7 will automatically update `project-context.md` (if implementation introduces new architectural patterns or conventions)
- Continuous use of agentic-delivery will naturally keep the document current
- Simple "use if exists" logic is more reliable

### Stage 2: Doc Scan (Part of Phase 0)

Dispatch doc-scanner subagent using `doc-scanner-prompt.md` to generate project context.

**调用方式：**
```
Task tool:
  description: "Scan project context"
  prompt: [content from doc-scanner/doc-scanner-prompt.md]
  subagent_type: "Explore"
```

**Strategy:** "Trust but Verify" - Read external docs as claims, verify critical assertions (tech stack, architecture) via code sampling, trust verified info while discarding contradictions.

**Process:**
1. Pre-check: Document freshness (skip if > 1 year old)
2. Read & Verify: Sample 10-20 files to validate claims
3. Fallback: Code-only scan if docs contradicted/missing
4. Generate: `project-context.md` with verification status (✅ Verified / ⚠️ Partial / ❌ Contradicted)

**Input:** Project root path (current working directory)
**Output:** `docs/project-context.md` (with verification markers, **in current directory**)
**Cost:** ~300 tokens (verified) or ~2000 tokens (code-only fallback)
**Lifecycle:** Dispose after completion.

**Path Note:** File is generated at `./docs/project-context.md` relative to current working directory, NOT in subdirectory projects.

**See:** `doc-scanner/doc-scanner-prompt.md` for complete dispatch template

**⚠️ SUBAGENT LIFECYCLE REMINDER:**
- **Codex:** After doc-scanner returns, immediately call `close_agent(agent_id)`
- **Claude Code:** Auto-closed when Task returns (no action needed)
- See `runtime-policies.md` § Subagent Lifecycle and `references/codex-tools.md` for details

### Stage 3: Requirement Analysis & Design

**Prerequisite:**
- ✅ Phase 0 complete, `docs/project-context.md` is ready

Execute brainstorming process (main agent, NOT subagent):

1. **Read `project-context.md`** (mandatory first step)
   - Extract tech stack, architectural patterns, code conventions
   - Follow these conventions during design
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

**Stage 5A: Review Track Selection (NEW - Smart Routing)**

BEFORE dispatching any implementers, analyze each task to assign appropriate review track:

1. **Read all tasks** from implementation plan
2. **For each task**, analyze:
   - Number of files affected
   - Estimated lines changed
   - Whether it creates new APIs
   - Whether it modifies architecture
   - Domain scope (frontend/backend/both)
   - Security sensitivity
3. **Assign track** using review-fix-strategy criteria:
   - Fast Track: Code Quality only (skips spec review)
   - Standard Track: Spec + Code Quality (default)
   - Heavy Track: Spec + Code + Integration
4. **Log assignments** with reasoning
5. **Store track metadata** for use in Stage 6

**Implementation (orchestrator executes inline):**

```python
# 伪代码 - 主 agent 在 Stage 5 开始时执行此逻辑
def assign_review_track(task):
    """
    分析任务特征，分配合适的 review track。
    返回: (track_name, reasoning)
    """
    # 提取任务元数据
    files = task.get("affected_files", [])
    lines = task.get("estimated_lines", 0)
    description = task.get("description", "")

    # 分析信号
    has_api = "API" in description or "interface" in description or "endpoint" in description
    domains = detect_domains(files)  # 检测涉及的领域 (frontend/backend/database)
    is_sensitive = any(keyword in description.lower()
                      for keyword in ["auth", "payment", "security", "credential", "token"])
    has_architecture_change = any(keyword in description.lower()
                                  for keyword in ["architecture", "pattern", "refactor", "redesign"])

    # Heavy Track（最高优先级）
    if len(domains) >= 2:
        return "Heavy", f"Cross-domain: {', '.join(domains)}"
    if has_architecture_change:
        return "Heavy", "Architecture change detected"
    if is_sensitive:
        return "Heavy", "Security-sensitive operation"

    # Fast Track（仅当非常简单）
    if len(files) == 1 and lines < 50 and not has_api:
        return "Fast", "Simple utility (1 file, <50 lines, no API)"

    # 默认：Standard Track
    return "Standard", "Typical feature work"

def detect_domains(files):
    """检测文件涉及的领域"""
    domains = set()
    for file in files:
        if any(pattern in file for pattern in ["frontend/", "ui/", "components/", "pages/"]):
            domains.add("frontend")
        if any(pattern in file for pattern in ["backend/", "api/", "server/", "services/"]):
            domains.add("backend")
        if any(pattern in file for pattern in ["database/", "db/", "models/", "schema/"]):
            domains.add("database")
    return list(domains)

# 主流程：对每个任务执行分析
for task in implementation_plan.tasks:
    track, reason = assign_review_track(task)
    task.metadata["review_track"] = track
    log(f"Task {task.id} ({task.name}): {track} Track - {reason}")
```

**Example track assignment output:**

```markdown
**Review Track Assignments:**

Task 1: Add validation helper → Fast Track
  - Single file (utils/validation.ts), ~35 lines
  - No API changes, pure utility
  - Review: Code Quality only

Task 2: User authentication API → Standard Track
  - 3 files (routes/controller/service), ~150 lines
  - New REST API endpoints
  - Review: Spec Compliance → Code Quality

Task 3: Payment integration → Heavy Track
  - Cross-domain (frontend checkout form + backend payment processor)
  - Security-sensitive (handles payment credentials)
  - Review: Spec Compliance → Code Quality → Integration
```

**Stage 5B: Orchestrator responsibilities:**
1. Extract ALL tasks with full text from plan (do NOT make subagent read plan file)
2. Route by dependency: independent tasks → parallel, dependent tasks → sequential
3. Construct minimum-context prompt per subagent (see Minimum Context §below)
4. Include assigned review track in task metadata for Stage 6

**Implementer receives:**
- Task full text (copy-pasted, not file path)
- Relevant spec fragment only (not entire design doc)
- Working directory
- Interface definitions from prerequisite tasks (if dependent)

**Status handling:**

| Status | Action |
|--------|--------|
| DONE | Proceed to Stage 6 review (using assigned track) |
| DONE_WITH_CONCERNS | Evaluate concerns, then review or address first |
| NEEDS_CONTEXT | Provide missing context, re-dispatch |
| BLOCKED | Assess: context gap → supplement; too complex → split task; plan wrong → back to Stage 4 |

**REQUIRED SUB-SKILL:** Use subagent-driven-development skill for dispatch pattern.

**⚠️ SUBAGENT LIFECYCLE REMINDER:**
- **After each implementer returns:** Close immediately (Codex: `close_agent(agent_id)`; Claude Code: auto)
- **Before next dispatch:** Run pre-dispatch cleanup check (see `references/codex-tools.md`)
- **Failure to close promptly** → slot exhaustion → dispatch failures → emergency cleanup required
- All implementer subagents are stateless; re-dispatch is cheap if needed again

**After parallel batch completes:**
If multiple implementers ran in parallel, verify their commits merge cleanly:
1. Check for merge conflicts (lock files, barrel exports, auto-generated files)
2. Resolve mechanical conflicts (main agent, not subagent)
3. Run project test suite to verify integration
4. If non-trivial conflicts exist, the plan incorrectly classified tasks as parallel — fix plan or escalate

**ENFORCEMENT:** See `runtime-policies.md` § Required Subagents for failure handling and violation detection.

### Stage 6: Test-Driven Verification

**Core principle:** 用客观的测试结果（tests pass + coverage）替代主观的 Code Review

**Verification strategies based on task track:**
- Fast Track: Test Verification only（简单任务）
- Standard Track: Test Verification + Spec Check（典型任务）
- Heavy Track: Test Verification + Spec Check + Integration（复杂任务）

**Why Test-Driven:**
- Speed: ~30s vs 2-5 min (4-10x faster)
- Objectivity: Pass/fail based on metrics, not opinions
- Repeatability: Same input → same result
- Traceability: Test reports + coverage data as evidence

---

#### Strategy A: Fast Track (Test Verification Only)

**When to use:** Tasks assigned Fast Track in Stage 5A (single file < 50 lines, pure utility)

**Flow:**
```
Implementer returns DONE (with Test Verification Data)
       │
       ▼
  Main Agent extracts data from implementer report:
    - Test execution output
    - Coverage report (actual vs target)
    - Linter output
    - Checkpoint commit SHA
       │
       ▼
  Dispatch Test Verification Agent
    Input:
      - Task description (from plan)
      - Coverage targets (from plan Test Strategy)
      - Test expectations (from plan Test Strategy)
      - Implementer's test data
      - Checkpoint commit SHA
       │
       ▼
  Test Verification Agent returns decision:
    ├─ approve → Mark task complete → Doc Sync → next task
    ├─ fix_tests → Dispatch Fix Agent → re-verify
    ├─ fix_coverage → Dispatch Fix Agent → re-verify
    └─ fix_linter → Dispatch Fix Agent → re-verify
```

**Cost:** 1 subagent call (test-verification-agent)
**Max rounds:** 1 (simple tasks should pass quickly)

#### Strategy B: Standard Track (Spec Check + Test Verification)

**When to use:** Tasks assigned Standard Track in Stage 5A (multi-file, new APIs, business logic)

**Flow:**
```
Implementer returns DONE (with Test Verification Data)
       │
       ▼
  Main Agent performs Spec Compliance Check (inline, not subagent):
    - Read task requirements
    - Read implementer report (what they claim they built)
    - Compare: all requirements met? any extra features?
       │
       ├─ All requirements met → proceed to test verification
       └─ Missing/extra → Dispatch Fix Agent → restart from spec check
       │
       ▼
  Dispatch Test Verification Agent
    (same as Fast Track)
       │
       ▼
  Test Verification Agent returns decision:
    ├─ approve → Mark task complete → Doc Sync → next task
    └─ fix_* → Dispatch Fix Agent → re-verify
```

**Cost:** 0-1 subagent calls (test-verification-agent; spec check is inline)
**Max rounds:** 2
**Why inline spec check:** Spec check is fast (read requirements list), no need for subagent overhead

#### Strategy C: Heavy Track (Spec + Test Verification + Integration)

**When to use:** Tasks assigned Heavy Track in Stage 5A (cross-domain, architecture changes, security-sensitive)

**Flow:**
```
Implementer returns DONE
       │
       ▼
  Main Agent: Inline Spec Check
       │
       ├─ Pass → continue
       └─ Fail → Fix → restart
       │
       ▼
  Dispatch Test Verification Agent
       │
       ├─ approve → continue to integration check
       └─ fix_* → Fix → re-verify
       │
       ▼
  (If multi-domain) Dispatch Integration Reviewer
    Input:
      - Design spec (API contract section)
      - Backend task git diffs
      - Frontend task git diffs
    Check:
      - API contract consistency
      - Data flow completeness
      - Error handling alignment
       │
       ├─ verified → Mark task complete → Doc Sync
       └─ issues → Dispatch Fix Agent → re-verify integration
```

**Cost:** 1-2 subagent calls (test-verification + integration)
**Max rounds:** 3

---

## How Orchestrator Prepares Test Verification Agent Input

**Step 1: Extract from Plan (Task N's Test Strategy)**
```python
# Pseudo-code
task = read_implementation_plan()["Task N"]
test_strategy = task["Test Strategy"]

coverage_targets = {
    "line": test_strategy["Coverage Targets"]["Line"],      # e.g., 80
    "branch": test_strategy["Coverage Targets"]["Branch"],  # e.g., 75
    "function": test_strategy["Coverage Targets"]["Function"] # e.g., 90
}

test_expectations = test_strategy["Test Expectations"]  # List of expected tests
```

**Step 2: Extract from Implementer Report**
```python
implementer_report = implementer_subagent.result

test_data = {
    "test_execution_output": implementer_report["Test Verification Data"]["Test Execution Output"],
    "coverage_report": implementer_report["Test Verification Data"]["Coverage Report"],
    "linter_output": implementer_report["Test Verification Data"]["Linter Output"],
    "checkpoint_commit": implementer_report["Checkpoint Commit"]
}

checklist = implementer_report["Pre-Submission Checklist Results"]
```

**Step 3: Construct Test Verification Agent Prompt**
```python
test_verification_prompt = f"""
You are verifying the test results for Task {task_id}: {task_name}

## Task Description
{task["description"]}

## Coverage Targets (from Plan)
- Line: ≥ {coverage_targets["line"]}%
- Branch: ≥ {coverage_targets["branch"]}%
- Function: ≥ {coverage_targets["function"]}%

## Test Expectations (from Plan)
{test_expectations}

## Implementer's Test Data
### Test Execution Output:
{test_data["test_execution_output"]}

### Coverage Report:
{test_data["coverage_report"]}

### Linter Output:
{test_data["linter_output"]}

### Checkpoint Commit:
{test_data["checkpoint_commit"]}

## Implementer's Checklist:
{checklist}

[... rest of prompt from test-verification-agent-prompt.md ...]
"""

dispatch_test_verification_agent(prompt=test_verification_prompt)
```

## Handling Test Verification Agent Decisions

**Decision: `approve`**
```python
if decision == "approve":
    # Mark task complete
    update_task_status(task_id, "completed")

    # Dispatch doc-syncer
    dispatch_doc_syncer(
        task_name=task_name,
        changed_files=implementer_report["Files Changed"],
        review_summary="Test Verification passed (N rounds)"
    )

    # Proceed to next task
    next_task()
```

**Decision: `fix_tests`**
```python
if decision == "fix_tests":
    # Issue: tests failed or incomplete
    # Dispatch Fix Agent to fix tests

    fix_prompt = f"""
    Task: {task_name}

    Issue: Test Verification Agent reported:
    {decision["issues"]}

    Fix needed:
    - Missing tests: {decision["verification_results"]["test_completeness"]["missing"]}
    - Failed tests: {decision["verification_results"]["test_execution"]["failed"]}

    Your job:
    1. Add missing tests OR fix failing tests
    2. Run tests to verify they pass
    3. Report back with updated test output
    """

    dispatch_fix_agent(prompt=fix_prompt, severity="Important")

    # After fix:
    # Re-dispatch Test Verification Agent with updated data
    re_verify()
```

**Decision: `fix_coverage`**
```python
if decision == "fix_coverage":
    # Issue: coverage below targets
    gaps = decision["verification_results"]["coverage"]

    fix_prompt = f"""
    Task: {task_name}

    Issue: Coverage gaps:
    - Line: {gaps["line"]["actual"]}% < {gaps["line"]["target"]}% (gap: {gaps["line"]["gap"]})
    - Branch: {gaps["branch"]["actual"]}% < {gaps["branch"]["target"]}% (gap: {gaps["branch"]["gap"]})
    - Function: {gaps["function"]["actual"]}% < {gaps["function"]["target"]}% (gap: {gaps["function"]["gap"]})

    Your job:
    1. Add tests to cover uncovered lines/branches/functions
    2. Run tests with --coverage
    3. Report back with updated coverage report
    """

    dispatch_fix_agent(prompt=fix_prompt, severity="Important")
    re_verify()
```

**Decision: `fix_linter`**
```python
if decision == "fix_linter":
    # Issue: linter errors
    linter_result = decision["verification_results"]["linter"]

    fix_prompt = f"""
    Task: {task_name}

    Issue: Linter errors: {linter_result["errors"]}

    Linter output:
    {linter_result["output"]}

    Your job:
    1. Fix all linter errors
    2. Run linter to verify 0 errors
    3. Report back with clean linter output
    """

    dispatch_fix_agent(prompt=fix_prompt, severity="Minor")
    re_verify()
```

**Max rounds exceeded:**
```python
round_count = count_verification_rounds(task_id)

if round_count > max_rounds[track]:
    # Fast Track: 1 round
    # Standard Track: 2 rounds
    # Heavy Track: 3 rounds

    escalate_to_user(
        task_id=task_id,
        issue="Test verification failed after {round_count} rounds",
        attempts=all_verification_attempts,
        recommendation="Manual review needed"
    )
```

---

**Commit strategy:**
- Implementer creates a checkpoint commit after implementation and local test verification
- Each verification-driven fix creates a separate fix commit
- A task is complete only after Test Verification passes (and Spec Check if applicable)

**After verification passes:** Dispatch doc-syncer subagent using `doc-syncer-prompt.md` to update `docs/<project>/<feature>/changelog.md`.

**调用方式：**
```
Task tool:
  description: "Update changelog"
  prompt: [content from doc-syncer/doc-syncer-prompt.md with filled parameters]
  subagent_type: "general-purpose"
```

**See:** `doc-syncer/doc-syncer-prompt.md` for complete dispatch template

**⚠️ SUBAGENT LIFECYCLE REMINDER:**
- **After test-verification-agent returns:** Close immediately (Codex: `close_agent(agent_id)`; Claude Code: auto)
- **Before next dispatch:** Pre-dispatch cleanup check
- **Agent lifecycle:** Test-verification-agent → close → (if needed) Fix agent → close → Doc syncer → close
- **Integration reviewer:** Close after integration report processed
- See `runtime-policies.md` § Subagent Lifecycle for enforcement rules

**After all tasks complete:**
- If plan contains frontend + backend tasks → Dispatch Integration Reviewer subagent
- Integration Reviewer checks: API contract consistency, data flow completeness, error handling alignment
- If issues found → Fix → Re-verify
- If verified → Proceed to Stage 7

**REQUIRED SUB-SKILL:** Use review-fix-strategy for fix decisions.

---

## Test-Driven Verification vs Traditional Code Review

**Why we switched:**

| Dimension | Traditional Code Review | Test-Driven Verification |
|-----------|------------------------|--------------------------|
| **Speed** | 2-5 min per task | ~30s per task (4-10x faster) |
| **Cost** | 2-4 subagent calls | 1 subagent call (50-75% reduction) |
| **Objectivity** | Subjective ("code looks good") | Objective (tests pass + coverage ≥ threshold) |
| **Repeatability** | Different reviewers → different results | Same input → same result |
| **Evidence** | Text opinions | Test reports + coverage data |
| **Verification** | Human reads code | Machine runs tests |

**What we gained:**
- ✅ Faster verification (30s vs 2-5 min)
- ✅ Objective pass/fail criteria
- ✅ Complete reproducibility
- ✅ Traceable evidence (test outputs)
- ✅ Coverage metrics

**What we traded:**
- ❌ Lost subjective code architecture review (now rely on tests + linter)
- ❌ Lost security vulnerability detection (now rely on tests + linter security plugins)
- ⚠️ Dependent on test quality (if tests are bad, verification passes bad code)

**Mitigation for trade-offs:**
- Use comprehensive linter with security plugins
- Require high coverage thresholds (≥80% line, ≥75% branch)
- Plan stage ensures Test Strategy is complete
- Spec check (Standard/Heavy tracks) ensures requirements met

## Legacy: Traditional Code Review (Deprecated)

**Status:** ⚠️ Deprecated - kept for backward compatibility only

The following review strategies are **no longer recommended**:
- Unified Reviewer (spec + code quality in one subagent)
- Sequential Two-Stage Review (spec reviewer → code quality reviewer)

**When to use (rare cases):**
- Project has no test infrastructure (no Jest/Vitest/Pytest)
- Task is not testable (e.g., pure documentation changes)
- User explicitly requests traditional review

**Prompts:** See `subagent-driven-development/unified-reviewer-prompt.md` (deprecated)

### Stage 7: Summary + Project Context Update

Main agent compiles delivery report:

- Requirement type and total task count
- Which tasks ran in parallel
- Review rounds and fix count
- All artifacts produced (docs, files changed)
- File change list with descriptions
- Lessons learned or optimization opportunities

**NEW: Update project-context.md if needed**

After compiling the summary, check if implementation introduced changes that affect project context:

**Detection Signals:**

1. **New top-level directories** (high signal)
   - Example: new `src/middleware/`, `src/plugins/`, etc.
   - Indicates: new architectural layer introduced

2. **New major dependencies** (medium signal)
   - Example: added `zod`, `prisma`, `trpc`, etc.
   - Indicates: new tech stack components introduced

3. **New architectural patterns** (high signal)
   - Extract from `design-spec.md` and `changelog.md`
   - Example: introduced DI container, Event Sourcing, CQRS, etc.
   - Indicates: new architectural patterns established

4. **New code conventions** (medium signal)
   - Extract from review reports
   - Example: unified error handling pattern, API response format, naming conventions, etc.
   - Indicates: new code conventions clarified

**Update Logic:**

```python
# Pseudo-code
def update_project_context_if_needed():
    """
    Update project-context.md at Stage 7 summary (if needed)
    """
    updates = []

    # Signal 1: New top-level directories
    new_dirs = detect_new_top_level_directories()
    if new_dirs:
        updates.append({
            "section": "## Directory Structure",
            "type": "append",
            "content": format_new_directories(new_dirs)
        })

    # Signal 2: New major dependencies
    new_deps = detect_new_major_dependencies()
    if new_deps:
        updates.append({
            "section": "## Tech Stack",
            "type": "append",
            "content": format_new_dependencies(new_deps)
        })

    # Signal 3: New architectural patterns (extract from design-spec/changelog)
    new_patterns = extract_architectural_patterns()
    if new_patterns:
        updates.append({
            "section": "## Architecture",
            "type": "append",
            "content": format_new_patterns(new_patterns)
        })

    # Signal 4: New code conventions (extract from review reports)
    new_conventions = extract_code_conventions_from_reviews()
    if new_conventions:
        updates.append({
            "section": "## Coding Conventions",
            "type": "append",
            "content": format_new_conventions(new_conventions)
        })

    # If there are updates, apply to file
    if len(updates) > 0:
        log(f"Detected {len(updates)} project context changes")
        apply_updates_to_project_context(updates)
        log("✅ Updated project-context.md")
    else:
        log("✅ Project context requires no updates")

def apply_updates_to_project_context(updates):
    """
    Apply updates to project-context.md
    """
    content = read_file("docs/project-context.md")

    for update in updates:
        section = update["section"]
        new_content = update["content"]

        # Find corresponding section, append content
        if section in content:
            # Append to end of section (before next ##)
            content = append_to_section(content, section, new_content)
        else:
            # Section doesn't exist, create new section
            content += f"\n\n{section}\n\n{new_content}"

    write_file("docs/project-context.md", content)
```

**Update Strategy:**

- **Conservative principle**: Only update obvious, verifiable changes (not based on speculation)
- **Append first**: Most cases append new info, rarely modify existing content
- **Main agent completes**: No additional subagent needed, Stage 7 main agent judges and updates itself
- **User visible**: Display updated content in summary

**Example Output:**

```
## Delivery Summary

### Execution Overview
- Requirement type: Large feature
- Total tasks: 5
- Parallel execution: Task 2, 3
- Review rounds: 8 total

### Stage outputs
... (regular summary) ...

### Project Context Update
✅ Detected the following changes and updated project-context.md:

1. **New directory structure**
   - `src/middleware/` — New middleware layer
   - `src/plugins/` — New plugin system

2. **New tech stack**
   - `zod` (v3.22.0) — Schema validation
   - `trpc` (v10.45.0) — Type-safe API

3. **New architectural patterns**
   - Dependency Injection — Using InversifyJS container
   - Plugin Architecture — Supports dynamic plugin loading

See `docs/project-context.md` for details (git diff shows specific changes)
```

**Why at Stage 7?**
- Only after implementation is complete do we know what changes were truly introduced
- Only after review passes can we confirm changes are reasonable and worth preserving
- Centralized updates are easier to track and rollback than distributed updates

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
| Doc Scanner (doc-scanner-prompt.md) | Project root, scan targets, output path | User requirements, design docs |
| Spec Reviewer | Design doc path, review checklist | Project code, plan |
| Plan Reviewer | Plan path, design doc path | Code details |
| Implementer (backend) | Task text, backend conventions, API schema | Frontend code/conventions, other tasks |
| Implementer (frontend) | Task text, frontend conventions, API schema | Backend code/conventions, other tasks |
| Spec Compliance Reviewer | Task requirements, implementer report, git diff (checkpoint commit) | Other tasks, global context |
| Code Quality Reviewer | Git diff (BASE_SHA..HEAD_SHA), task description | Requirements doc, other tasks |
| Fix Agent | Review report + relevant files from the latest checkpoint (Minor) or spec only (Critical) | Other tasks, global context |
| Doc Syncer (doc-syncer-prompt.md) | Task description, changed files, changelog template, date | Code details, review process |

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
- **Skip subagent cleanup after processing result (Codex: missing `close_agent`)**
- **Dispatch new subagent without pre-dispatch cleanup check**
- **Keep completed subagents open "just in case" (all subagents are stateless)**
- **Wait until dispatch fails before cleanup (reactive, not proactive)**
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
