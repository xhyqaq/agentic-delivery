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
| Incremental Feature | 依赖已有功能、扩展现有模块、用户明确提到"在XX基础上/基于XX/扩展XX" | Incremental Pipeline (复用目录) |
| Large Feature | 全新功能、多模块、架构影响、无依赖现有功能 | Full Pipeline (新建目录) |
| Small Change | Single file/function, clear scope | Fast Path (Stage 5-7 only) |
| Bug Fix | Explicit error behavior, needs root cause analysis | Debug Path |

**Intent Recognition Process:**

1. **Check for existing feature dependency (FIRST):**
   ```python
   # Pseudo-code
   existing_features = glob("docs/*/")  # ["docs/skills-market/", "docs/user-auth/", ...]

   for feature_dir in existing_features:
       feature_name = extract_name(feature_dir)  # "skills-market"

       # Signal 1: User explicitly mentions existing feature
       if feature_name in user_request.lower():
           return classify_as_incremental(feature_name, feature_dir)

       # Signal 2: Semantic similarity (keywords, domain overlap)
       if is_semantically_related(user_request, feature_dir):
           # Ask user to confirm
           confirm = ask_user(f"Does this relate to existing '{feature_name}' feature?")
           if confirm:
               return classify_as_incremental(feature_name, feature_dir)

   # No dependency found → New feature or Fast path
   return classify_by_scope(user_request)  # Large Feature or Small Change
   ```

2. **Incremental vs New Feature decision tree:**
   ```
   用户需求提到现有功能名称？
   ├─ Yes → Incremental Feature
   └─ No → 检查语义相关性
       ├─ 高相关（同一领域、扩展现有 API）→ Ask user → Incremental Feature
       └─ 低相关（全新领域）→ Large Feature
   ```

**Cross-session reuse:**

**For New Features (Large Feature path):**
- Create new directory: `docs/<new-feature>/`
- Generate fresh `design-spec.md`
- Generate fresh `implementation-tracker.md`

**For Incremental Features (Incremental Pipeline):**
- Reuse directory: `docs/<existing-feature>/`
- Update existing `design-spec.md` (Living Document strategy - see Stage 3)
- Create timestamped tracker: `implementation-tracker-YYYY-MM-DD.md`
- Check existing artifacts:
  - `design-spec.md` exists → **UPDATE** in Stage 3 (not skip!)
  - Previous trackers exist → create new timestamped tracker
  - Previous tracker has unchecked tasks → warn user about incomplete work

## Full Pipeline (Large Feature + Incremental Feature)

### Phase 0: Project Context Preparation (Blocking Prerequisite)

**Trigger:** Large feature path OR Incremental feature path (both use full pipeline)

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

**Path fork based on intent type:**

#### Path A: New Feature (Large Feature)

Execute brainstorming process (main agent, NOT subagent):

1. **Read `project-context.md`** (mandatory first step)
   - Extract tech stack, architectural patterns, code conventions
   - Follow these conventions during design
2. Follow the `brainstorming` questioning strategy to clarify requirements
   - Batch independent questions when possible
   - Ask sequentially only when later questions depend on earlier answers
3. Propose 2-3 approaches with trade-offs, give recommendation
4. Present design in sections, get user approval per section
5. **Write to** `docs/<project>/<new-feature>/design-spec.md`
6. Dispatch Spec Reviewer subagent → review loop (max 3 rounds)
7. User reviews written spec before proceeding

#### Path B: Incremental Feature

Execute brainstorming process with existing context:

1. **Read `project-context.md`** (mandatory first step)
2. **Read existing `design-spec.md`** (understand current design)
   - Understand existing architecture, API contracts, data models
   - Identify which sections will be affected by the increment
3. Follow the `brainstorming` questioning strategy for the increment
   - Focus questions on the new/changed parts
   - Reference existing design where relevant
4. Propose 2-3 approaches for the increment (considering existing architecture)
5. Present design in sections, get user approval per section
6. **Update `design-spec.md` using Living Document Strategy** (see below)
7. Dispatch Spec Reviewer subagent → review loop (max 3 rounds)
8. User reviews updated spec before proceeding

**Living Document Strategy (Incremental Feature):**

```python
# Pseudo-code
def update_design_spec(existing_spec_path, increment_design):
    """
    Update design-spec.md for incremental feature

    Returns: path to design doc (may be new file if exception case)
    """
    existing_spec = read_file(existing_spec_path)
    existing_size = count_lines(existing_spec)

    # Exception 1: Breaking change → Create version doc
    if increment_design.is_breaking_change:
        # Breaking change examples:
        # - REST API → GraphQL migration
        # - Database schema redesign
        # - Authentication system overhaul
        new_spec_path = existing_spec_path.replace(".md", "-v2.md")
        write_file(new_spec_path, increment_design.full_content)
        log(f"⚠️ Breaking change detected, created {new_spec_path}")
        log(f"   Old design preserved in {existing_spec_path}")
        return new_spec_path

    # Exception 2: Large independent subsystem + doc too large → Create module doc
    if increment_design.is_large_subsystem and existing_size > 1000:
        # Large subsystem examples:
        # - New recommendation engine (>5 sections, independent)
        # - Payment processing module
        # - Analytics dashboard subsystem
        module_name = increment_design.module_name
        module_spec_path = f"docs/<feature>/design-spec-{module_name}.md"
        write_file(module_spec_path, increment_design.full_content)

        # Add reference to main spec
        append_to_spec(existing_spec_path,
            f"\n\n## {module_name.title()} Module\n\n"
            f"See [design-spec-{module_name}.md](./design-spec-{module_name}.md) for details.\n"
        )

        log(f"✅ Created module doc {module_spec_path}")
        log(f"   Reference added to main spec")
        return module_spec_path

    # Default: Update in place (Living Document - 活文档)
    updated_spec = merge_increment(existing_spec, increment_design)
    write_file(existing_spec_path, updated_spec)

    # Update changelog
    update_changelog(
        path=f"docs/<feature>/changelog.md",
        entry={
            "date": today(),
            "type": "Feature Increment",
            "summary": increment_design.summary,
            "sections_modified": increment_design.affected_sections
        }
    )

    log(f"✅ Updated {existing_spec_path} (Living Document)")
    log(f"   Changelog updated: changelog.md")
    return existing_spec_path

def merge_increment(existing_spec, increment_design):
    """
    Merge increment into existing spec

    Strategy:
    - If increment extends existing section → append to that section
    - If increment adds new capability → add new section
    - If increment modifies existing behavior → update relevant sections
    """
    # Parse existing spec structure
    sections = parse_markdown_sections(existing_spec)

    for new_section in increment_design.sections:
        if new_section.name in sections:
            # Extend existing section
            sections[new_section.name] = extend_section(
                sections[new_section.name],
                new_section.content
            )
        else:
            # Add new section
            sections[new_section.name] = new_section.content

    return rebuild_markdown(sections)

def is_breaking_change(increment_design):
    """
    Detect breaking changes that require version doc

    Signals:
    - Migration keywords (REST→GraphQL, SQL→NoSQL)
    - Incompatible API changes
    - Data model redesign
    - Authentication system changes
    """
    breaking_keywords = [
        "migrate", "redesign", "overhaul", "replace",
        "incompatible", "breaking change"
    ]

    description = increment_design.summary.lower()
    return any(keyword in description for keyword in breaking_keywords)

def is_large_subsystem(increment_design):
    """
    Detect large independent subsystem

    Criteria:
    - >5 major sections
    - Standalone functionality
    - Minimal coupling to existing code
    """
    return (
        len(increment_design.sections) > 5 and
        increment_design.independence_score > 0.7  # 70% independent
    )
```

**Document Strategy Decision Criteria:**

| Condition | Strategy | Output | Use Case |
|-----------|----------|--------|----------|
| Breaking change (REST→GraphQL, schema redesign) | Version Doc | `design-spec-v2.md` | Major architectural shift |
| Large subsystem (>5 sections) + doc >1000 lines | Module Doc | `design-spec-<module>.md` | Independent large module |
| Normal increment (新字段、新API、功能增强) | Living Doc | Update `design-spec.md` | **DEFAULT** |
| Tiny change (single field, small patch) | Fast Path | Skip design stage | Trivial changes |

**Why Living Document is default:**
- ✅ Single source of truth (design-spec.md is always current)
- ✅ Git provides history (can diff/revert any change)
- ✅ changelog.md provides human-readable summary
- ✅ Cross-session recovery simple (read one file, not N files)
- ✅ Prevents document fragmentation
- ✅ Aligns with how code evolves (files change, not multiply)

**REQUIRED SUB-SKILL:** Follow brainstorming skill.
`agentic-delivery` routes into this stage; `brainstorming` owns the detailed questioning method.

### Stage 4: Implementation Plan

**Path fork based on intent type:**

#### Path A: New Feature (Large Feature)

Main agent writes detailed plan:

1. Map file structure: which files to create/modify, exact paths
2. Decompose into bite-sized tasks (2-5 min each)
3. Each task: file paths, steps, test commands, expected results, commit message
4. Mark task dependencies (parallel vs sequential)
5. Dispatch Plan Reviewer subagent → if issues found, fix and escalate to user

**Output:** `docs/<project>/<feature>/implementation-tracker.md`

#### Path B: Incremental Feature

Main agent writes incremental plan:

1. **Analyze existing implementation** (read current codebase in affected areas)
   - Understand current file structure
   - Identify which files need modification
   - Locate integration points
2. Map file structure: which files to **modify** (prioritize) vs **create** (if needed)
   - Prefer modifying existing files over creating new ones
   - Create new files only when necessary
3. Decompose into bite-sized tasks (2-5 min each)
4. Mark dependencies with existing code
5. Dispatch Plan Reviewer subagent → if issues found, fix and escalate to user

**Output:** `docs/<project>/<feature>/implementation-tracker-<YYYY-MM-DD>.md`

**Why timestamped tracker for increments:**
- Each increment is a distinct batch of work (independent delivery unit)
- Keeps historical record of what was done when
- Avoids confusion between old and new tasks
- Allows parallel increments on different dates (multiple teams/sessions)
- Enables audit trail (which features were added in which increment)

**Format example:** `implementation-tracker-2026-03-26.md`

**Parallel constraint (HARD RULE):**
> Tasks sharing ANY file MUST be sequential. Only tasks with zero file overlap may be parallel.
> This is decided at plan time, NOT execution time.

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
- `implementation-tracker.md` exists and contains tasks
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
for task in implementation_tracker.tasks:
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

## Commit Verification Checkpoint (RECOMMENDED)

**When:** Immediately after implementer returns DONE, before extracting test data

**Purpose:** Verify implementer's checkpoint commit is real and valid

**User Preference:** Verification failures are **non-blocking** (log warning, continue)

### Verification Commands

Run these checks to validate the checkpoint commit:

```bash
# Step 1: Verify commit exists
git cat-file -t <commit_sha>
# Expected output: "commit"
# If fails: commit does not exist

# Step 2: Verify commit message format
git log -1 --format=%B <commit_sha>
# Check:
# - Starts with conventional type: "feat:" | "fix:" | "refactor:" | "test:" | "docs:"
# - Contains task reference (optional but recommended)

# Step 3: Verify files in commit
git diff-tree --no-commit-id --name-only -r <commit_sha>
# Check: all files reported by implementer are in commit

# Step 4: Verify commit is on current branch
git branch --contains <commit_sha>
# Check: current branch is in the list
```

### Handling Verification Results

**If ALL checks pass:**
```
✅ Checkpoint commit verified: <commit_sha>
[Log success and proceed to Test Verification]
```

**If ANY check fails (non-blocking warning mode):**
```
⚠️ Checkpoint commit verification warning

Task: [Task N: name]
Reported commit: <commit_sha>
Issues detected:
  - [❌ Commit does not exist]
  - [❌ Invalid commit message format]
  - [❌ Missing expected files: file1.ts, file2.ts]
  - [❌ Commit not on current branch]

[Continue to Test Verification despite warnings]
[User can review and fix issues later if needed]
```

### Example Orchestrator Code

```python
# Pseudo-code for orchestrator
def verify_checkpoint_commit(checkpoint_commit, expected_files):
    """
    Verify checkpoint commit (non-blocking).

    Returns:
        (warnings: List[str]) - Empty list if all checks pass
    """
    warnings = []

    # Check 1: Commit exists
    result = run_command(f"git cat-file -t {checkpoint_commit}")
    if result.returncode != 0:
        warnings.append(f"Commit {checkpoint_commit} does not exist")
        return warnings  # Cannot continue other checks

    # Check 2: Commit message format
    commit_msg = run_command(f"git log -1 --format=%B {checkpoint_commit}").stdout
    valid_types = ["feat:", "fix:", "refactor:", "test:", "docs:", "chore:"]
    if not any(commit_msg.startswith(t) for t in valid_types):
        warnings.append("Commit message missing conventional type prefix")

    # Check 3: Files in commit
    changed_files = run_command(
        f"git diff-tree --no-commit-id --name-only -r {checkpoint_commit}"
    ).stdout.strip().split("\n")

    missing_files = [f for f in expected_files if f not in changed_files]
    if missing_files:
        warnings.append(f"Expected files not in commit: {missing_files}")

    # Check 4: Commit on current branch
    branches = run_command(f"git branch --contains {checkpoint_commit}").stdout
    current_branch = run_command("git branch --show-current").stdout.strip()
    if current_branch not in branches:
        warnings.append(f"Commit not on current branch ({current_branch})")

    return warnings

# Usage in Stage 6
checkpoint_commit = implementer_result["Checkpoint Commit"]
expected_files = [f["path"] for f in implementer_result["Files Changed"]]

warnings = verify_checkpoint_commit(checkpoint_commit, expected_files)

if warnings:
    log_warning(f"⚠️ Commit verification issues: {warnings}")
    # Continue to Test Verification (non-blocking)
else:
    log(f"✅ Checkpoint commit verified: {checkpoint_commit}")

# Proceed to Test Verification regardless of warnings
```

---

## How Orchestrator Prepares Test Verification Agent Input

**Step 1: Extract from Plan (Task N's Test Strategy)**
```python
# Pseudo-code
task = read_implementation_tracker()["Task N"]
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

    # Update implementation-tracker.md after Test Verification approves
    update_implementation_tracker_after_verification(
        task_id=task_id,
        task_name=task_name,
        implementer_report=implementer_report,
        verification_result=verification_result
    )
```

### Update Implementation Plan After Test Verification

After Test Verification Agent approves, update the implementation-tracker.md:

```python
# Pseudo-code for orchestrator
def update_implementation_tracker_after_verification(task_id, task_name, implementer_report, verification_result):
    """
    Update implementation-tracker.md after Test Verification approves.

    Args:
        task_id: Task number (e.g., 1)
        task_name: Task name from plan
        implementer_report: Full implementer output
        verification_result: Test Verification Agent decision
    """
    # Extract data
    checkpoint_commit = implementer_report["Checkpoint Commit"]
    test_cases = implementer_report["Test Verification Data"]["Test Names"]
    test_file = extract_test_file_from_report(implementer_report)
    verified_timestamp = current_timestamp()  # Format: "YYYY-MM-DD HH:MM"

    # Build the completion section
    test_case_bullets = "\n  ".join([f"- ✓ {tc}" for tc in test_cases])
    completion_text = f"""
  **Tests:** (added after completion)
  {test_case_bullets}
  **Test file:** `{test_file}`
  **Checkpoint Commit:** {checkpoint_commit[:7]}
  **Verified:** {verified_timestamp}
"""

    # Find the task in plan
    plan_content = read_file("docs/<project>/<feature>/implementation-tracker.md")

    # Update task checkbox from [ ] to [x]
    updated_content = plan_content.replace(
        f"- [ ] Task {task_id}: {task_name}",
        f"- [x] Task {task_id}: {task_name}"
    )

    # Insert completion section after the task's Test Strategy section
    # (Find the insertion point right before the next task or end of file)
    task_pattern = f"### Task {task_id}:"
    next_task_pattern = f"### Task {task_id + 1}:"

    insertion_point = find_insertion_point(updated_content, task_pattern, next_task_pattern)
    updated_content = insert_text_at(updated_content, insertion_point, completion_text)

    # Write back to plan
    write_file("docs/<project>/<feature>/implementation-tracker.md", updated_content)

    log(f"✅ Task {task_id} marked complete in implementation-tracker.md")
    log(f"   Checkpoint: {checkpoint_commit[:7]}")
    log(f"   Verified: {verified_timestamp}")

# Example usage in Stage 6
if verification_result["overall_status"] == "PASS":
    # First dispatch doc-syncer
    dispatch_doc_syncer(...)

    # Then update implementation plan
    update_implementation_tracker_after_verification(
        task_id=current_task_id,
        task_name=current_task_name,
        implementer_report=implementer_result,
        verification_result=verification_result
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

- Requirement type (New Feature / Incremental Feature / Small Change / Bug Fix)
- Base feature (if incremental)
- Total task count
- Which tasks ran in parallel
- Review rounds and fix count
- All artifacts produced (docs, files changed)
- File change list with descriptions
- Lessons learned or optimization opportunities

**For Incremental Features: Update changelog.md**

After implementation completes, add entry to `docs/<feature>/changelog.md`:

```markdown
## [Increment] YYYY-MM-DD - <Short Description>

### Added
- New capability 1
- New capability 2

### Changed
- Changed behavior in X
- Enhanced Y to support Z

### Design Changes
- Updated sections in design-spec.md:
  - Section A: <what changed>
  - Section B: <what changed>

### Implementation
- Tasks completed: N
- Review rounds: M
- Files modified: X
- Files created: Y
- Implementation tracker: `implementation-tracker-2026-03-26.md`

### Commits
- feat: <commit 1 summary> (abc123)
- feat: <commit 2 summary> (def456)
```

**Changelog format follows [Keep a Changelog](https://keepachangelog.com/) standard:**
- **Added**: new features
- **Changed**: changes in existing functionality
- **Deprecated**: soon-to-be removed features
- **Removed**: removed features
- **Fixed**: bug fixes
- **Security**: security fixes

**Why changelog for increments:**
- Provides human-readable summary of what changed
- Links to implementation tracker for technical details
- Tracks evolution of the feature over time
- Helps future sessions understand feature history
- Supplements Git commit history with context

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
