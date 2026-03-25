# Doc Scanner Subagent Prompt Template

**LIFECYCLE NOTICE:**
- 这是无状态的一次性 subagent
- 返回项目上下文后自动关闭
- 不要保持活动状态

---

**Purpose:** Generate project context document for new features using "Trust but Verify" strategy

**When to dispatch:** At Stage 2 (Doc Scan) of agentic-delivery workflow for large features

---

## How to Dispatch

```
Task tool:
  description: "Scan project context"
  prompt: |
    You are scanning a project to generate project context documentation.

    ## Input
    - Project root: {project_root}
    - Scan targets: {scan_targets}
    - Output file: {output_path}

    ## Your Job

    Follow the complete workflow defined in doc-scanner/SKILL.md:

    **Phase 1: Pre-check (< 50ms)**
    - Check if external docs exist and are fresh (< 1 year old)
    - If no docs OR docs > 1 year → skip to Phase 3

    **Phase 2: Read & Verify (< 500ms)**
    - Read documentation claims from README.md, CLAUDE.md, docs/
    - Extract key claims (Tech Stack, Architecture, Conventions)
    - Verify P0 claims (Tech Stack) via code sampling:
      - Sample 10 random files from src/
      - Calculate match rate
      - ✅ Verified (>70%), ⚠️ Partial (30-70%), ❌ Contradicted (<30%)
    - Verify P1 claims (Architecture) if possible
    - Trust P2 claims (Conventions) unless obviously contradicted

    **Phase 3: Fallback - Code-Only Scan (if docs missing/outdated/contradicted)**
    - Scan project structure with `tree -L 3`
    - Infer tech stack from package.json, lock files, imports
    - Sample code conventions from 20 random files
    - Generate project-context.md with "✅ Inferred from Code" markers

    **Phase 4: Generate Output**
    - Use Format A (Verified Documentation) if Phase 2 succeeded
    - Use Format B (Code-Only) if Phase 3 was used
    - Include verification status for each section
    - Mark uncertainties as ⚠️

    See doc-scanner/SKILL.md for complete details and output templates.

    ## Output Format

    Return JSON:
    {
      "status": "DONE",
      "output_file": "docs/<project>/project-context.md",
      "verification_status": "✅ Verified" | "⚠️ Partial" | "❌ Code-only",
      "token_cost": 300-2000,
      "key_findings": {
        "tech_stack": "React 18 + TypeScript (✅ 90% verified)",
        "architecture": "3-layer pattern (⚠️ exists but not strictly enforced)",
        "conventions": "camelCase functions (✅ 95% observed)"
      }
    }

  subagent_type: "Explore"
```

---

## Input Parameters (from orchestrator)

| Parameter | Type | Example |
|-----------|------|---------|
| project_root | string | "/Users/xhy/Desktop/my-project" |
| scan_targets | array | ["README.md", "CLAUDE.md", "AGENTS.md", "docs/"] |
| output_path | string | "docs/my-project/project-context.md" |

---

## Output Contract

**Success (DONE):**
```json
{
  "status": "DONE",
  "output_file": "docs/<project>/project-context.md",
  "verification_status": "✅ Verified" | "⚠️ Partial" | "❌ Code-only",
  "token_cost": 300-2000
}
```

**The generated file contains:**
- Tech Stack section (with verification markers)
- Architecture section (if applicable)
- Coding Conventions section
- Directory Structure
- External Documentation Status

---

## Quality Gates

- All critical claims (Tech Stack) verified with match rate > 70% OR marked as ⚠️/❌
- Uncertain claims marked with ⚠️ symbol
- Contradicted claims discarded and rescanned from code (❌ → ✅ Inferred)
- Output file written to specified path
- Token cost within expected range (300-2000 tokens)

---

## See Also

- **Full workflow details:** `doc-scanner/SKILL.md`
- **Trust but Verify strategy:** `doc-scanner/SKILL.md` Phase 2
- **Output format templates:** `doc-scanner/SKILL.md` Phase 4
