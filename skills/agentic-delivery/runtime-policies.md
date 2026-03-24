# Runtime Policies

These are HARD RULES that cannot be bypassed. They complement SKILL.md with enforcement details.

## Required Subagents (Cannot Be Skipped)

### Stage 4: Code Implementation

**Rule:** When executing implementation plan with 2+ tasks, you MUST dispatch subagents. No exceptions.

#### Parallel Implementation Requirement

When the plan includes distinct slices (e.g., frontend + backend), you MUST:
1. Dispatch dedicated subagents with explicit ownership boundaries
2. Execute independent tasks concurrently (in parallel)
3. Never implement inline first "to test" or "to save time"

**Example violation:**
```
❌ "后端我会先一次性补齐 skills 子域的最小闭环..."
   (Backend I will complete in one go...)

   This is implementing inline. FORBIDDEN.
```

**Correct approach:**
```
✅ "我现在派发两个 subagent：
   - Backend Implementer: 处理 tasks 1-5
   - Frontend Implementer: 处理 tasks 6-10"

   Uses spawn_agent or Task tool to dispatch both.
```

#### Failure Handling

If subagent dispatch fails (tool unavailable, platform limitation):

1. **STOP execution immediately**
2. **Report the blocker to the user:**
   ```
   "Cannot proceed with Stage 4: subagent dispatch is unavailable.

    Error: [specific tool error]

    Required capability: spawn_agent or Task tool
    Current platform: [Codex/Claude Code/etc]

    Please verify tool availability or run in supported environment."
   ```
3. **DO NOT:**
   - Silently fall back to single-agent mode
   - Implement tasks yourself as a "workaround"
   - Continue with Stage 5 review
   - Skip to Stage 6 summary

**Rationale:** Single-agent inline implementation defeats the core purposes of this workflow:
- ❌ No parallel execution (slower delivery)
- ❌ No context isolation (higher error risk)
- ❌ No specialized review quality (lower code quality)
- ❌ Violates user expectation of agentic-delivery pattern

If subagents are unavailable, the workflow is **blocked** — not degraded.

### Stage 5: Review Loop

**Rule:** Fresh subagent per review round. Never reuse reviewers across rounds.

**Rule:** Review order is mandatory:
1. Spec Compliance Review FIRST
2. Code Quality Review SECOND (only if #1 passes)

### Stage 1: Doc Scan

**Rule:** Must dispatch doc-scanner subagent for large features. Main agent does NOT do document scanning inline.

## Verification Checks

Before entering each stage, verify prerequisites:

### Before Stage 4:
```python
# Logical checks (not actual code):
assert stage >= 3, "Must complete Stage 3 (Implementation Plan) first"
assert file_exists("docs/<project>/<feature>/implementation-plan.md"), "No plan file"
assert can_use_tool("spawn_agent") or can_use_tool("Task"), "Cannot dispatch subagents"
assert len(tasks) > 0, "Empty plan"

if len(tasks) >= 2:
    # Check for parallelizable tasks
    frontend_tasks = [t for t in tasks if "frontend" in t.scope]
    backend_tasks = [t for t in tasks if "backend" in t.scope]

    if len(frontend_tasks) > 0 and len(backend_tasks) > 0:
        print("✓ Detected frontend + backend slices → MUST dispatch parallel subagents")
```

### Before Stage 5:
```python
assert implementer_returned_DONE, "Cannot review incomplete implementation"
assert not implementing_inline, "If you wrote code yourself, you violated Stage 4 rules"
```

## Cross-Platform Compatibility

### Codex
- Use `spawn_agent` tool
- Subagents receive `session_id` and `parent_id`
- Check tool availability with `codex status` or inspect tool list

### Claude Code
- Use `Task` tool with `subagent_type` parameter
- Subagents are stateless (single message return)
- Tool always available in standard environments

### Cursor
- Limited subagent support
- May need to report blocker for Stage 4 with 3+ tasks
- Small changes (Fast Path) can proceed

## Enforcement Philosophy

**Why strict enforcement:**
1. Prevents silent quality degradation
2. Makes failures visible and debuggable
3. Ensures user gets the workflow they requested
4. Maintains consistency across sessions

**If a rule feels too strict for a specific case:**
1. DO NOT bypass it silently
2. Report to user with clear reasoning
3. Let user decide to modify workflow or proceed differently
4. Document the exception in session notes

## Common Violations and Fixes

| Violation | Detection | Fix |
|-----------|-----------|-----|
| Implementing inline in Stage 4 | Search session for `exec_command(sed/rg)` during Stage 4 | Abort and restart with subagent dispatch |
| Skipping subagent for "simple" task | Plan has 2+ tasks but only 1 subagent dispatched | Dispatch dedicated subagent per task |
| Reusing reviewer across rounds | Same `agent_id` in multiple review rounds | Spawn fresh reviewer per round |
| Not checking tool availability | No verification before Stage 4 | Add explicit check at Stage 3→4 transition |

## Audit Trail

Each session should be auditable for compliance:

**Required evidence in session log:**
- [ ] Stage 0: Intent classification decision
- [ ] Stage 3: implementation-plan.md written
- [ ] Stage 4: spawn_agent/Task calls (count = task count)
- [ ] Stage 4: NO exec_command calls that modify code
- [ ] Stage 5: spawn_agent/Task calls for reviewers (2× per task)
- [ ] Stage 6: Summary includes "N tasks ran in parallel"

**Red flag in log:**
- exec_command with sed/rg/git during Stage 4
- Comments like "我会先一次性补齐" (I will complete in one go)
- Task count > 1 but only 1 or 0 subagents spawned
