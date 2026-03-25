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
    - Project root: {project_root}  (当前工作目录)
    - Scan targets: {scan_targets}
    - Output file: {output_path}  (默认: docs/project-context.md)

    ## Your Job

    **At the start, output progress message:**
    ```
    Starting project context scan...
    Phase 1: Checking existing documentation
    Phase 2: Verifying tech stack information
    Phase 3: Generating project context document
    ```

    Follow the complete workflow defined in doc-scanner/SKILL.md:

    **Phase 1: Pre-check (< 50ms)**
    - [Output] "Phase 1/3: Checking existing documentation..."
    - Check if external docs exist and are fresh (< 1 year old)
    - If no docs OR docs > 1 year → skip to Phase 3

    **Phase 2: Read & Verify (< 500ms)**
    - [Output] "Phase 2/3: Verifying tech stack information (sampling 10-20 files)..."
    - Read documentation claims from README.md, CLAUDE.md, docs/
    - Extract key claims (Tech Stack, Architecture, Conventions)
    - Verify P0 claims (Tech Stack) via code sampling:
      - Sample 10 random files from src/
      - Calculate match rate
      - ✅ Verified (>70%), ⚠️ Partial (30-70%), ❌ Contradicted (<30%)
    - Verify P1 claims (Architecture) if possible
    - Trust P2 claims (Conventions) unless obviously contradicted

    **Phase 3: Fallback - Code-Only Scan (if docs missing/outdated/contradicted)**
    - [Output] "Phase 3/3: Inferring project information from code..."
    - Scan project structure with `tree -L 3`
    - Infer tech stack from package.json, lock files, imports
    - Sample code conventions from 20 random files
    - Generate project-context.md with "✅ Inferred from Code" markers

    **Phase 4: Generate Output**
    - [Output] "Generating project context document: docs/project-context.md"
    - Use Format A (Verified Documentation) if Phase 2 succeeded
    - Use Format B (Code-Only) if Phase 3 was used
    - Include verification status for each section
    - Mark uncertainties as ⚠️
    - **IMPORTANT**: Write to the exact path provided in {output_path}
      - Default is `docs/project-context.md` in current working directory
      - NOT in subdirectory projects (e.g., NOT `frontend/docs/`)
      - Create `docs/` directory if it doesn't exist

    See doc-scanner/SKILL.md for complete details and output templates.

    **After completion, output summary:**
    ```json
    {
      "status": "DONE",
      "output_file": "docs/project-context.md",
      "verification_status": "✅ Verified",
      "phases_executed": ["Phase 1: Pre-check", "Phase 2: Verify", "Phase 4: Generate"],
      "token_cost": 350,
      "key_findings": {
        "tech_stack": "React 18 + TypeScript (✅ 90% verified)",
        "architecture": "3-layer pattern (⚠️ exists but not strictly enforced)",
        "conventions": "camelCase functions (✅ 95% observed)"
      },
      "estimated_time": "45 seconds"
    }
    ```

    ## Output Format

    Return JSON:
    {
      "status": "DONE",
      "output_file": "docs/project-context.md",
      "verification_status": "✅ Verified" | "⚠️ Partial" | "❌ Code-only",
      "phases_executed": ["list", "of", "phases"],
      "token_cost": 300-2000,
      "key_findings": {
        "tech_stack": "React 18 + TypeScript (✅ 90% verified)",
        "architecture": "3-layer pattern (⚠️ exists but not strictly enforced)",
        "conventions": "camelCase functions (✅ 95% observed)"
      },
      "estimated_time": "30-90 seconds"
    }

  subagent_type: "Explore"
```

---

## Input Parameters (from orchestrator)

| Parameter | Type | Example | Default |
|-----------|------|---------|---------|
| project_root | string | $(pwd) | 当前工作目录 |
| scan_targets | array | ["README.md", "CLAUDE.md", "AGENTS.md", "docs/"] | 标准文档列表 |
| output_path | string | "docs/project-context.md" | **docs/project-context.md**（当前目录下） |

**重要说明：**
- `output_path` 默认为 `docs/project-context.md`，相对于当前工作目录
- **不是** `docs/<project>/project-context.md`（避免嵌套到子项目）
- 如果 `docs/` 不存在，doc-scanner 会自动创建

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
