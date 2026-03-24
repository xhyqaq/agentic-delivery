# Changelog

All notable changes to agentic-delivery will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-03-24

### Added

#### Core System
- **Intent Recognition** (Stage 0): Classify tasks as Large Feature / Small Change / Bug Fix
- **7-Stage Full Pipeline**: Doc Scan → Requirement Analysis → Design → Plan → Implementation → Review → Summary
- **3 Workflow Paths**: Full Pipeline, Fast Path (small changes), Debug Path (bug fixes)
- **Cross-Session Recovery**: Reuse existing design-spec.md and implementation-plan.md

#### Skills (12 total)

**Custom Skills (4):**
- `agentic-delivery` — Main orchestration skill with 7-stage workflow
- `doc-scanner` — Project structure extraction and context generation
- `doc-syncer` — Automated changelog updates after each task
- `review-fix-strategy` — Tiered fix decision framework (Minor/Important/Critical)

**Adapted from Superpowers (8):**
- `brainstorming` — Requirement analysis and design with iterative review
- `writing-plans` — Implementation plan authoring with task decomposition
- `subagent-driven-development` — Task-by-task execution with two-stage review
- `dispatching-parallel-agents` — Parallel task orchestration
- `requesting-code-review` — Code review template and workflow
- `systematic-debugging` — 4-phase debugging methodology
- `verification-before-completion` — Evidence-before-claims verification principle
- `finishing-a-development-branch` — Branch completion and cleanup workflow

#### Architecture Features
- **Minimum Context Principle**: Each subagent receives only task-specific context
- **Stateless Subagents**: Fresh instance per dispatch, no context pollution
- **File Conflict Prevention**: Parallel constraint enforced at plan stage
- **Document Standardization**: Fixed templates for all persistent docs
- **Model Selection Strategy**: Use cheapest model per role (Fast/Standard/Capable)
- **Two-Stage Review**: Spec Compliance → Code Quality
- **Tiered Fix Strategy**: Minor/Important (patch) vs Critical (reimplement), max 3 rounds

#### Platform Support
- **Claude Code**: Plugin installation via `.claude-plugin/plugin.json`
- **Codex**: Native skill discovery via `~/.agents/skills/` symlink
- **Platform Adaptation**: Tool name mapping for Codex compatibility (see `skills/agentic-delivery/references/codex-tools.md`)

#### Documentation
- Complete design document (11 chapters, 742 lines)
- Installation guides for Claude Code and Codex
- README with architecture highlights and quick start
- MIT License

### Notes

- This is the initial release based on multi-agent orchestration design principles
- 8 of 12 skills adapted from [obra/superpowers](https://github.com/obra/superpowers)
- External dependencies (not included): `test-driven-development`, `executing-plans`, `using-git-worktrees` — install separately if needed

[1.0.0]: https://github.com/xhyqaq/agentic-delivery/releases/tag/v1.0.0
