# Migration Guide: Traditional Code Review → Test-Driven Verification

## Overview

As of 2026-03-25, the agentic-delivery system switched from **Traditional Code Review** to **Test-Driven Verification**. This guide explains the migration.

## Key Changes

### What Changed

| Aspect | Before | After |
|--------|--------|-------|
| **Verification method** | Subjective code review (reviewer reads code) | Objective test verification (agent checks test results) |
| **Stage 6 flow** | Implementer → Spec Reviewer → Code Quality Reviewer | Implementer → (Spec Check) → Test Verification Agent |
| **Speed** | 2-5 min per task | ~30s per task |
| **Evidence** | Text opinions | Test reports + coverage data |

### Why We Changed

**Problems with Traditional Code Review:**
1. **Slow:** Reading code takes 2-5 minutes
2. **Subjective:** Different reviewers give different feedback
3. **Not reproducible:** Can't re-run review with same result
4. **No evidence:** Just text opinions, no data

**Benefits of Test-Driven Verification:**
1. **Fast:** Running tests takes ~30 seconds
2. **Objective:** Pass = tests pass + coverage ≥ threshold
3. **Reproducible:** Same input → same result
4. **Evidence:** Test outputs + coverage reports

## For Users

### If you're starting a new feature

✅ **No action needed** - follow the normal agentic-delivery flow. Test-Driven Verification is now the default.

### If you have an in-progress feature (using old system)

**Option 1: Continue with old system (Deprecated)**
- Set flag: `use_test_verification: false` in project config
- System will use legacy Unified Reviewer
- ⚠️ Not recommended, will be removed in future

**Option 2: Migrate to new system (Recommended)**
- Complete current tasks with old system
- For new tasks in the same feature:
  1. Update `implementation-tracker.md` to add **Test Strategy** section for remaining tasks
  2. Ensure Implementer provides **Test Verification Data** in reports
  3. System will automatically use Test Verification Agent

## For Implementers (Subagents)

### Old Report Format
```markdown
**Status:** DONE

**What you implemented:** ...

**Files Changed:** ...

**Self-Review:** [subjective comments]
```

### New Report Format (Required)
```markdown
**Status:** DONE

**What you implemented:** ...

**Files Changed:** ...

**Checkpoint Commit:** abc123def

---

## Test Verification Data (NEW - Machine Verifiable)

**Test Execution Output:**
```
Tests: 5 passed, 5 total
✓ test1 (8ms)
✓ test2 (5ms)
...
```

**Coverage Report:**
```
File      | Line % | Branch % | Func %
----------|--------|----------|-------
file.ts   | 85.7%  | 80.0%    | 100%
Target    | 80%    | 75%      | 90%
```

**Linter Output:**
```
✓ No errors
⚠ 1 warning
```

---

## Pre-Submission Checklist Results

**Test Verification (AUTO-VERIFIED):**
- ✅ Tests pass: 5/5
- ✅ Coverage: 85.7% line, 80.0% branch
- ✅ Linter: 0 errors, 1 warning

[rest of checklist...]
```

## For Plan Writers

### Old Plan Format (No Test Strategy)
```markdown
### Task 3: Implement login API

**Steps:**
- [ ] Implement function
- [ ] Write tests
- [ ] Verify works
- [ ] Commit
```

### New Plan Format (Required)
```markdown
### Task 3: Implement login API

**Test Strategy:**
- Test Type: Integration
- Coverage Targets: Line ≥ 80%, Branch ≥ 75%, Function ≥ 90%
- Test Framework: Jest
- Test File: tests/api/login.test.ts
- Quality Gates: Linter 0 errors, Type check 100%

**Test Expectations:**
- ≥ 1 Happy Path test: valid credentials → return token
- ≥ 2 Edge Cases tests: empty password, non-existent user
- ≥ 2 Error Cases tests: expired token, malformed request

**Steps:**
- [ ] Write failing tests
- [ ] Implement function
- [ ] Verify tests pass
- [ ] Check coverage ≥ targets
- [ ] Run linter (0 errors)
- [ ] Commit
```

## Troubleshooting

### "My project doesn't have Jest/coverage tools"

**Short-term:** Use legacy code review (set `use_test_verification: false`)

**Long-term:** Set up test infrastructure:
1. Install Jest: `npm install --save-dev jest @jest/globals`
2. Add to package.json: `"test": "jest --coverage"`
3. Update plan to use Test-Driven Verification

### "Test Verification Agent rejected my task"

**Decision: `fix_tests`**
- Issue: Tests failed or incomplete
- Solution: Fix failing tests or add missing tests

**Decision: `fix_coverage`**
- Issue: Coverage < targets (e.g., 70% < 80%)
- Solution: Add tests to cover uncovered lines/branches

**Decision: `fix_linter`**
- Issue: Linter errors > 0
- Solution: Fix linter errors

### "How do I know what tests to write?"

Read your task's **Test Expectations** in `implementation-tracker.md`. It lists:
- Happy path tests (≥1)
- Edge case tests (≥2)
- Error case tests (≥2)

## Timeline

- **2026-03-25:** Test-Driven Verification becomes default
- **2026-06-30:** Legacy code review deprecated warning added
- **2026-09-30:** Legacy code review removed (breaking change)

## Feedback

Report issues or questions to: [GitHub Issues](https://github.com/xhy/agentic-delivery/issues)
