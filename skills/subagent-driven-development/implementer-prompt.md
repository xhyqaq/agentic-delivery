# Implementer Subagent Prompt Template

**LIFECYCLE NOTICE (for dispatcher):**
- This is a **stateless, single-use** subagent
- **After this subagent returns its result:**
  - **Codex:** You MUST call `close_agent(agent_id)` immediately
  - **Claude Code:** Auto-closed on return (no action needed)
- **Do NOT keep this subagent active** after processing result
- If you need this role again later: re-dispatch a fresh instance

---

Use this template when dispatching an implementer subagent.

```
Task tool (general-purpose):
  description: "Implement Task N: [task name]"
  prompt: |
    You are implementing Task N: [task name]

    ## Task Description

    [FULL TEXT of task from plan - paste it here, don't make subagent read file]

    ## Context

    [Scene-setting: where this fits, dependencies, architectural context]

    ## Before You Begin

    If you have questions about:
    - The requirements or acceptance criteria
    - The approach or implementation strategy
    - Dependencies or assumptions
    - Anything unclear in the task description

    **Ask them now.** Raise any concerns before starting work.

    ## Your Job

    Once you're clear on requirements:
    1. Implement exactly what the task specifies
    2. Write tests (following TDD if task says to)
    3. Verify implementation works
    4. **Create a checkpoint commit** (MANDATORY)
    5. Self-review (see below)
    6. Report back

    Work from: [directory]

    ## Creating the Checkpoint Commit

    **CRITICAL: Use HEREDOC format to ensure special characters are handled correctly**

    ```bash
    git add [files]
    git commit -m "$(cat <<'EOF'
    feat: [Brief description of what you implemented]

    - [Detailed change 1]
    - [Detailed change 2]
    - [Detailed change 3]

    Task: [Task name from plan]
    EOF
    )"
    ```

    **Commit message requirements:**
    - Type prefix: `feat:` (new feature) | `fix:` (bug fix) | `refactor:` (code improvement) | `test:` (tests only) | `docs:` (documentation)
    - Brief description: ≤ 50 characters, imperative mood ("add", not "added")
    - Detailed changes: Bullet list of what changed and why
    - Task reference: Copy task name from plan for traceability

    **Why HEREDOC (`<<'EOF'`):**
    - Prevents shell special characters ($, `, \, ") from breaking commit message
    - Supports multi-line messages cleanly
    - Ensures literal string (no variable expansion)

    **Example:**
    ```bash
    git commit -m "$(cat <<'EOF'
    feat: implement user authentication API

    - Add JWT token generation and validation
    - Implement login endpoint with email/password
    - Add token refresh mechanism
    - Handle expired token errors

    Task: Implement login API (Task 3)
    EOF
    )"
    ```

    **After committing:**
    - Verify commit was created: `git log -1 --oneline`
    - Record the commit SHA (first 7-9 characters)
    - Report the SHA in your output under "Checkpoint Commit"

    The commit is a checkpoint for review and rollback, not final approval of the task.

    **While you work:** If you encounter something unexpected or unclear, **ask questions**.
    It's always OK to pause and clarify. Don't guess or make assumptions.

    ## Code Organization

    You reason best about code you can hold in context at once, and your edits are more
    reliable when files are focused. Keep this in mind:
    - Follow the file structure defined in the plan
    - Each file should have one clear responsibility with a well-defined interface
    - If a file you're creating is growing beyond the plan's intent, stop and report
      it as DONE_WITH_CONCERNS — don't split files on your own without plan guidance
    - If an existing file you're modifying is already large or tangled, work carefully
      and note it as a concern in your report
    - In existing codebases, follow established patterns. Improve code you're touching
      the way a good developer would, but don't restructure things outside your task.

    ## When You're in Over Your Head

    It is always OK to stop and say "this is too hard for me." Bad work is worse than
    no work. You will not be penalized for escalating.

    **STOP and escalate when:**
    - The task requires architectural decisions with multiple valid approaches
    - You need to understand code beyond what was provided and can't find clarity
    - You feel uncertain about whether your approach is correct
    - The task involves restructuring existing code in ways the plan didn't anticipate
    - You've been reading file after file trying to understand the system without progress

    **How to escalate:** Report back with status BLOCKED or NEEDS_CONTEXT. Describe
    specifically what you're stuck on, what you've tried, and what kind of help you need.
    The controller can provide more context, re-dispatch with a more capable model,
    or break the task into smaller pieces.

    ## Before Reporting Back: Mandatory Pre-Submission Checklist

    YOU MUST complete this checklist before reporting DONE. Report each item's status in your response.

    **Spec Compliance Check:**
    - [ ] All requirements from task description are implemented (list each one: ✅ done / ⚠️ concern / ❌ missed)
    - [ ] No extra features added beyond spec (✅ clean / ⚠️ added minor / ❌ over-built)
    - [ ] All edge cases mentioned in spec are handled (list each one: ✅ handled / ❌ missed)
    - [ ] API contracts match spec exactly (field names, types, status codes)

    **Code Quality Check:**
    - [ ] No magic numbers (✅ all extracted to constants with clear names / ⚠️ found N)
    - [ ] No TODO/FIXME/XXX comments left in code (✅ clean / ❌ found N)
    - [ ] No debug code (console.log, print statements, commented code) (✅ clean / ❌ found N)
    - [ ] Naming matches project conventions (check existing code) (✅ consistent / ⚠️ unsure)
    - [ ] All new functions/classes have single, clear purpose (✅ yes / ⚠️ concern)
    - [ ] Error handling is complete (not just happy path) (✅ yes / ⚠️ partial)

    **Testing Check:**
    - [ ] All core paths tested (not just happy path) (✅ yes / ⚠️ partial)
    - [ ] Edge cases from spec have test coverage (✅ yes / ❌ missed N)
    - [ ] Tests actually PASS (not just written) (✅ all pass / ❌ N failures)
    - [ ] Test names clearly describe what they verify (✅ yes / ⚠️ some unclear)

    **Integration Check (if applicable):**
    - [ ] If task has dependencies, interfaces match (✅ verified / ⚠️ assumed / N/A)
    - [ ] If task provides APIs, response format matches spec (✅ exact match / ⚠️ close / N/A)
    - [ ] If task consumes APIs, error codes are handled (✅ all / ⚠️ some / N/A)

    **Format your checklist report like this:**
    ```
    ## Pre-Submission Checklist Results

    **Spec Compliance:**
    - ✅ Requirement 1: User authentication
    - ✅ Requirement 2: Token validation
    - ✅ No extra features
    - ✅ Edge case: expired token → handled
    - ✅ Edge case: malformed token → handled

    **Code Quality:**
    - ✅ No magic numbers (TOKEN_EXPIRY = 3600)
    - ✅ No debug code
    - ⚠️ Found 1 TODO (line 45: "optimize this loop") - marked as DONE_WITH_CONCERNS
    - ✅ Naming consistent with existing auth/ module
    - ✅ Single purpose: validateToken() only validates, not logs

    **Testing:**
    - ✅ 8/8 tests pass
    - ✅ Core paths: valid/invalid/expired/malformed tokens
    - ✅ Test names: "should reject expired token with 401"

    **Integration:**
    - ✅ AuthService.validateToken() matches interface from Task 1
    - ✅ Returns {valid: boolean, userId?: string} as per spec
    - N/A No APIs consumed
    ```

    **If you find issues during checklist:** Fix them now before reporting. If you cannot fix (e.g., architectural decision needed), report as DONE_WITH_CONCERNS or BLOCKED.

    **Why this matters:** Your self-review is the first line of defense. Catching issues now saves 2-3 review rounds later. Be honest - if you're unsure about something, flag it.

    ## Report Format

    When done, report:

    **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

    **What you implemented:** (brief summary)

    **Checkpoint Commit:** abc123def

    **Files Changed:**
    - src/user.ts (new, 120 lines)
    - src/user.test.ts (new, 180 lines)

    ---

    ## Test Verification Data (NEW - Machine Verifiable)

    **Test Execution Output:**
    ```
    Test Suites: 1 passed, 1 total
    Tests:       5 passed, 5 total
    Time:        1.2s

    ✓ should return user when valid ID (8ms)
    ✓ should handle empty database (5ms)
    ✓ should handle ID at boundary (6ms)
    ✓ should throw TypeError for invalid ID type (3ms)
    ✓ should throw RangeError for negative ID (4ms)
    ```

    **Coverage Report:**
    ```
    File      | Line % | Branch % | Func % | Status
    ----------|--------|----------|--------|--------
    user.ts   | 85.7%  | 80.0%    | 100%   | ✅ PASS
    Target    | 80%    | 75%      | 90%    |
    ```

    **Linter Output:**
    ```
    ✓ No errors
    ⚠ 1 warning:
      - user.ts:23 - Consider extracting magic number
    ```

    **How to Reproduce:**
    ```bash
    git checkout abc123def
    npm test src/user.test.ts --coverage
    npm run lint src/user.ts
    ```

    ---

    ## Pre-Submission Checklist Results

    **Test Verification (AUTO-VERIFIED):**
    - ✅ Tests pass: 5/5 ✓ (from test output above)
    - ✅ Coverage: 85.7% line, 80.0% branch (from coverage report above)
    - ⚠️ Linter: 1 warning (within threshold < 5)

    **Spec Compliance (MANUAL):**
    - ✅ Requirement 1: getUser function implemented
    - ✅ Requirement 2: Error handling for invalid inputs
    - ✅ No extra features

    **Manual Quality Checks:**
    - ✅ Naming consistent with existing codebase
    - ✅ Single responsibility principle followed

    **Issues or Concerns:**
    - ⚠️ One linter warning about magic number (minor, can fix later)

    **Status decision rules:**
    - **DONE:** All checklist items ✅, confident in work
    - **DONE_WITH_CONCERNS:** Mostly ✅ but found ⚠️ items that might need attention
    - **BLOCKED:** Found ❌ items that require architectural decisions or more context
    - **NEEDS_CONTEXT:** Missing information needed to complete checklist

    Never report DONE if you have ⚠️ or ❌ items without explaining them.
```
