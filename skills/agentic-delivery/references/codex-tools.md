# Codex Tool Mapping

Skills use Claude Code tool names. When you encounter these in a skill, use your platform equivalent:

| Skill references | Codex equivalent |
|-----------------|------------------|
| `Task` tool (dispatch subagent) | `spawn_agent` |
| Multiple `Task` calls (parallel) | Multiple `spawn_agent` calls |
| Task returns result | `wait` |
| Task completes automatically | `close_agent` to free slot |
| `TodoWrite` (task tracking) | `update_plan` |
| `Skill` tool (invoke a skill) | Skills load natively — just follow the instructions |
| `Read`, `Write`, `Edit` (files) | Use your native file tools |
| `Bash` (run commands) | Use your native shell tools |

## Subagent dispatch requires multi-agent support

Add to your Codex config (`~/.codex/config.toml`):

```toml
[features]
multi_agent = true
```

This enables `spawn_agent`, `wait`, and `close_agent` for skills like `dispatching-parallel-agents` and `subagent-driven-development`.

---

## Subagent Lifecycle Management

### Platform Differences

**Critical difference between platforms:**

| Platform | Dispatch | Wait for Result | Cleanup |
|----------|----------|----------------|---------|
| **Claude Code** | `Task(description="...", prompt="...")` | Blocks until complete (automatic) | **Auto-closes** when result returns |
| **Codex** | `spawn_agent(prompt="...")` returns `agent_id` | `wait(agent_id)` blocks until complete | **MUST call `close_agent(agent_id)`** explicitly |

**Key insight:** Claude Code's `Task` tool handles the full lifecycle automatically. Codex requires explicit cleanup via `close_agent` to free the slot.

---

### Cleanup Strategies

#### Codex: Explicit Cleanup Required

**You MUST:**
1. Track every `agent_id` returned by `spawn_agent`
2. After processing the result, immediately call `close_agent(agent_id)`
3. Before dispatching a new agent, close any completed agents first

**Why it matters:**
- Session has limited subagent slots (typically 5-10)
- Unclosed agents occupy slots even after they're done
- When slots fill up, new dispatches fail
- You'll be forced to manually clean up (reactive, not proactive)

**Pattern (Codex):**

```python
# Dispatch
implementer_id = spawn_agent(prompt="Implement Task 1...")
log(f"Dispatched implementer: {implementer_id}")

# Wait for result
result = wait(implementer_id)

# Process result
status, files, commit = parse_result(result)
update_tracking(status, files, commit)

# ⚠️ CRITICAL: Close immediately after processing
close_agent(implementer_id)
log(f"Closed implementer: {implementer_id}")

# Now you can dispatch the next agent
reviewer_id = spawn_agent(prompt="Review Task 1...")
# ... repeat the pattern
```

#### Claude Code: Auto-Cleanup

**No action needed:**
- `Task` tool automatically closes the subagent when it returns
- You don't need to (and can't) call any cleanup function
- The slot is freed immediately

**Pattern (Claude Code):**

```python
# Dispatch and wait (combined in one tool call)
result = Task(
    description="Implement Task 1",
    prompt="...",
    subagent_type="implementer"
)

# Process result
status, files, commit = parse_result(result)
update_tracking(status, files, commit)

# Subagent is already closed automatically
# Immediately dispatch next agent if needed
next_result = Task(description="Review Task 1", ...)
```

---

### Pre-dispatch Checklist (Codex Only)

**Before EVERY `spawn_agent` call:**

```python
# Check active subagent count
active_count = len(active_subagents)

# If any completed agents remain unclosed: close them
for agent_id in list(completed_subagents):
    try:
        close_agent(agent_id)
        log(f"Pre-dispatch cleanup: closed {agent_id}")
    except:
        pass  # Already closed

completed_subagents.clear()

# If approaching slot limit, emergency cleanup
if active_count >= (SLOT_LIMIT - 1):
    log("⚠️ Approaching slot limit, executing emergency cleanup")
    # Close all completed agents
    # (implementation depends on your tracking)

# Log pre-dispatch state
log(f"Pre-dispatch check: {active_count} active subagents")

# Now safe to dispatch
new_agent_id = spawn_agent(...)
```

---

### Post-processing Checklist (Codex Only)

**After EVERY `wait()` returns:**

```python
# 1. Extract all info from result
status = result.get("status")
files = result.get("files")
commit = result.get("commit")
concerns = result.get("concerns")

# 2. Update tracking
update_plan_file(task_id, status)
update_todo_list(task_id, "completed")

# 3. ⚠️ CLOSE AGENT IMMEDIATELY ⚠️
close_agent(agent_id)
log(f"Closed {role}: {agent_id}")

# 4. Remove from active list
del active_subagents[agent_id]
completed_subagents.append(agent_id)  # Audit trail

# 5. Decide next action
if status == "DONE":
    # Proceed to review
elif status == "BLOCKED":
    # Handle blocker
# ...
```

**Critical timing:** Close BEFORE deciding next action, not after. This ensures the slot is free before you need it again.

---

### Error Recovery (Codex)

**Scenario:** `spawn_agent` fails with "Session limit reached" or similar error.

**Root cause:** Completed subagents were not closed promptly.

**Recovery flow:**

```python
# 1. STOP current operation
log("🛑 Dispatch failed: Session slot limit reached")

# 2. Audit active subagents
log(f"Active subagents: {list(active_subagents.keys())}")
log(f"Completed (should be closed): {completed_subagents}")

# 3. Identify which are completed
# Completed = result already processed, not waiting
# In-progress = currently waiting for result

# 4. Close all completed subagents
closed_count = 0
for agent_id in list(active_subagents.keys()):
    # If this agent's result was already processed:
    if agent_is_completed(agent_id):
        try:
            close_agent(agent_id)
            closed_count += 1
            log(f"Emergency cleanup: closed {agent_id}")
        except Exception as e:
            log(f"Failed to close {agent_id}: {e}")

log(f"Emergency cleanup: closed {closed_count} subagents")

# 5. Clear tracking
active_subagents = {k: v for k, v in active_subagents.items()
                    if not agent_is_completed(k)}
completed_subagents.clear()

# 6. Retry original dispatch
try:
    new_agent_id = spawn_agent(original_prompt)
    log(f"Retry successful: {new_agent_id}")
except Exception as e:
    # 7. Still fails → escalate to user
    log(f"⛔ Retry failed: {e}")
    log("Escalating to user with diagnostic info")
    raise
```

---

### Example Implementation (Codex)

**Full lifecycle management for a task:**

```python
# Session-level tracking
active_subagents = {}       # {agent_id: (role, dispatch_time)}
completed_subagents = []    # [agent_id, ...] (audit trail)

def dispatch_and_close(role, prompt):
    """
    Dispatch a subagent, wait for result, process it, and close immediately.

    Args:
        role: Subagent role ("implementer", "reviewer", etc.)
        prompt: Full prompt text for the subagent

    Returns:
        Processed result dict
    """
    # Pre-dispatch cleanup
    cleanup_completed_subagents()

    # Dispatch
    agent_id = spawn_agent(prompt=prompt)
    active_subagents[agent_id] = (role, now())
    log(f"✅ Dispatched {role}: {agent_id}")

    try:
        # Wait for result
        result = wait(agent_id)

        # Process result
        info = parse_result(result)
        log(f"📥 {role} returned: {info.get('status')}")

        # Update tracking
        if info.get('status') == 'DONE':
            update_plan(info)

        return info

    finally:
        # ⚠️ ALWAYS close in finally block (even if error)
        try:
            close_agent(agent_id)
            completed_subagents.append(agent_id)
            del active_subagents[agent_id]
            log(f"🚮 Closed {role}: {agent_id}")
        except Exception as e:
            log(f"⚠️ Failed to close {agent_id}: {e}")

def cleanup_completed_subagents():
    """Close all agents in completed list."""
    for agent_id in completed_subagents:
        try:
            close_agent(agent_id)
            log(f"🧹 Pre-dispatch cleanup: closed {agent_id}")
        except:
            pass  # Already closed, ignore
    completed_subagents.clear()

# Usage:
try:
    # Task 1: Implementation
    impl_result = dispatch_and_close("implementer", "Implement Task 1...")

    # Task 1: Spec review
    spec_result = dispatch_and_close("spec_reviewer", "Review Task 1 spec compliance...")

    # Task 1: Code quality review
    quality_result = dispatch_and_close("code_reviewer", "Review Task 1 code quality...")

    log(f"✅ Task 1 complete. Active subagents: {len(active_subagents)}")

except Exception as e:
    log(f"⛔ Error: {e}")
    # Emergency cleanup
    cleanup_completed_subagents()
    raise
```

---

### Best Practices

**Do:**
- ✅ Close immediately after processing result
- ✅ Use `try/finally` to ensure cleanup even on error
- ✅ Track active and completed agents
- ✅ Run pre-dispatch cleanup check
- ✅ Log every dispatch and close operation

**Don't:**
- ❌ Keep completed agents "just in case" — re-dispatch is cheap
- ❌ Wait until dispatch fails before cleaning up
- ❌ Skip cleanup for "the last dispatch" — always clean
- ❌ Forget to close if subagent returns an error

**Remember:**
- All subagents are **stateless** — no benefit to keeping them alive
- Re-dispatching a fresh agent is **cheap** — just a new prompt
- Cleanup is **mandatory**, not optional
- Failure to clean up is a **policy violation** that blocks execution

---

### Audit Trail

**Required logging for Codex sessions:**

```
[Stage 5 Start] Platform: Codex, Cleanup mode: explicit, Slot limit: 8
[Task 1] ✅ Dispatched implementer: agent_001
[Task 1] 📥 implementer returned: DONE
[Task 1] 🚮 Closed implementer: agent_001
[Task 1] 🧹 Pre-dispatch cleanup: completed list cleared
[Task 1] ✅ Dispatched spec_reviewer: agent_002
[Task 1] 📥 spec_reviewer returned: PASS
[Task 1] 🚮 Closed spec_reviewer: agent_002
[Task 1] 🧹 Pre-dispatch cleanup: completed list cleared
[Task 1] ✅ Dispatched code_reviewer: agent_003
[Task 1] 📥 code_reviewer returned: PASS
[Task 1] 🚮 Closed code_reviewer: agent_003
[Task 2] 🧹 Pre-dispatch cleanup: completed list cleared, 0 active
[Task 2] ✅ Dispatched implementer: agent_004
...

[Stage 5 End] Total dispatched: 12, Total closed: 12, Unclosed: 0, Status: ✅ COMPLIANT
```

---

### Troubleshooting

**Problem:** `spawn_agent` fails with "Session limit reached"

**Diagnosis:**
```python
# Check how many agents are still active
print(f"Active: {len(active_subagents)}")  # Should be 0 or 1
print(f"Completed (should be empty): {completed_subagents}")

# If completed list is non-empty, cleanup wasn't executed
```

**Fix:**
```python
# Close all completed agents
for agent_id in list(active_subagents.keys()):
    if agent_is_completed(agent_id):
        close_agent(agent_id)

# Clear tracking
active_subagents.clear()
completed_subagents.clear()

# Retry
spawn_agent(prompt)
```

**Problem:** Lost track of which agents are completed

**Prevention:**
```python
# Use explicit status tracking
subagent_status = {}  # {agent_id: "waiting" | "completed" | "closed"}

# Update status at each step:
agent_id = spawn_agent(...)
subagent_status[agent_id] = "waiting"

result = wait(agent_id)
subagent_status[agent_id] = "completed"

close_agent(agent_id)
subagent_status[agent_id] = "closed"
```
