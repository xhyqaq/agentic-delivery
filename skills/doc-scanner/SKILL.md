---
name: doc-scanner
description: "Use when starting a large feature and needing to understand project structure, conventions, and existing documentation before design work begins"
---

# Doc Scanner

Scan project structure and existing documentation to generate a standardized project context file. This enables subagents in later stages to receive accurate, minimal context about the project.

## When to Use

- Called by `agentic-delivery` at Stage 2 (Doc Scan)
- When `docs/<project>/project-context.md` does not exist or is outdated
- When entering a new project for the first time

## The Process

1. **Scan project structure**
   - Generate directory tree (depth 2-3 levels)
   - Detect tech stack from config files (`package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, etc.)
   - Identify package manager, build tool, test framework

2. **Read existing documentation**
   - `CLAUDE.md` (project root and `.claude/` directory)
   - `AGENTS.md`
   - `README.md`
   - `docs/*.md` (top-level docs)
   - Any architecture or contributing guides

3. **Extract from code if necessary**
   - Entry points (main files, index files)
   - Core module identification (by directory structure and imports)
   - Key dependency list (from lock files / manifests)

4. **Generate project-context.md**
   - Write to `docs/<project>/project-context.md`
   - Follow the template EXACTLY (see below)

## Output Template

The output MUST follow this exact section structure. Fill each section. If information is unavailable, write "Not detected" rather than omitting the section.

```markdown
# Project Context

## Basic Info
- **Project Name**:
- **Tech Stack**:
- **Package Manager**:
- **Build Tool**:
- **Test Framework**:
- **Test Command**:
- **Build Command**:
- **Lint Command**:

## Directory Structure
[Core directory tree, depth 2-3 levels only]

## Core Modules
| Module | Path | Responsibility |
|--------|------|----------------|
| ... | ... | ... |

## Coding Conventions
[Extracted from CLAUDE.md / AGENTS.md / existing code patterns]
- Naming conventions
- File organization patterns
- Import ordering
- Error handling patterns

## Known Constraints
[Deployment environment, compatibility requirements, performance requirements, etc.]

## Existing Documentation
| File | Summary |
|------|---------|
| CLAUDE.md | [one-line summary] |
| ... | ... |
```

## What NOT to Do

- Do NOT read every file in the project — scan structure, read docs and configs only
- Do NOT include generated files, node_modules, build artifacts in directory tree
- Do NOT infer conventions that aren't evidenced by existing code or docs
- Do NOT include sensitive information (secrets, credentials, API keys)

## Lifecycle

**Created by:** `agentic-delivery` main agent at Stage 2
**Disposed:** Immediately after `project-context.md` is written
**Output persists:** As file on disk, reusable across sessions
