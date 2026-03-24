# Token Monitor

**Purpose:** Track and report token usage across stages to quantify optimization effectiveness and identify bottlenecks.

## Monitoring Points

Track token counts at these key points:

1. **Stage 3 output:** Spec file size (design-spec.md)
2. **Stage 4 output:** Plan file size (implementation-plan.md)
3. **Stage 5 per-task:** Implementer input prompt size
4. **Stage 6 per-review:** Reviewer input prompt size (Spec Compliance + Code Quality)

## Estimation Method

Use rough estimation for token counts:
- **File-based:** `token_count ≈ file_size_in_chars / 4`
- **Prompt-based:** `token_count ≈ prompt_length_in_chars / 4`

This is approximate but sufficient for trend analysis and optimization tracking.

## Reporting Format

At Stage 7 (Project Summary), include a Token Usage Report:

```markdown
### Token Usage Report

| Stage | Component | Tokens | Notes |
|-------|-----------|--------|-------|
| 3 | Spec | 4,500 | design-spec.md |
| 4 | Plan | 8,000 | implementation-plan.md |
| 5 | Implementer (Task 1) | 1,200 | Input prompt |
| 5 | Implementer (Task 2) | 1,100 | Input prompt |
| 5 | Implementer (Task 3) | 1,150 | Input prompt |
| 6 | Spec Reviewer (Task 1) | 800 | Input prompt |
| 6 | Code Reviewer (Task 1) | 600 | git diff |
| 6 | Spec Reviewer (Task 2) | 850 | Input prompt |
| 6 | Code Reviewer (Task 2) | 550 | git diff |
| ... | ... | ... | ... |
| **Total Input** | | **35,000** | Estimated |

**Optimization opportunities:**
- Plan size could be reduced by 40% (see writing-plans optimization)
- Implementer inputs share 60% duplicate content (consider reference mechanism)
- Spec file is well-sized for current complexity
```

## Usage in agentic-delivery Skill

The main agentic-delivery orchestrator should:

1. **Track at each stage:**
   - After writing spec: estimate spec file tokens
   - After writing plan: estimate plan file tokens
   - Before each implementer dispatch: estimate prompt size
   - Before each reviewer dispatch: estimate prompt size

2. **Accumulate totals:**
   - Keep running total across all tasks
   - Separate by category (spec, plan, implementer, reviewer)

3. **Report at Stage 7:**
   - Include Token Usage Report in project summary
   - Highlight optimization opportunities
   - Compare to baseline if previous data available

## Benefits

- **Visibility:** Users can see token consumption patterns
- **Optimization tracking:** Quantify improvements from Phase 1/2/3 optimizations
- **Bottleneck identification:** Find where token costs are highest
- **ROI calculation:** Justify optimization efforts with cost savings

## Example Integration

In the Stage 7 summary, after listing all completed tasks:

```markdown
## Project Summary

All tasks completed successfully:
- ✅ Task 1: Implement authentication service
- ✅ Task 2: Add user profile endpoints
- ✅ Task 3: Create integration tests

### Token Usage Report
[Insert table above]

**Phase 1 optimization impact:**
- Baseline (pre-optimization): ~35,000 tokens
- Current execution: ~24,500 tokens
- **Savings: 30%** (~10,500 tokens, ~$0.21 per execution)
```

## Future Enhancements

- **Automated tracking:** Integrate with Claude Code's usage API (if available)
- **Historical trends:** Track token usage across multiple executions
- **Per-file breakdown:** Show which files consume most tokens
- **Model cost calculator:** Convert tokens to actual dollar costs
