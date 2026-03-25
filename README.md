# Agentic Delivery

**End-to-end delivery orchestration for coding agents** — A complete software development workflow built on composable skills and multi-agent architecture.

## What is Agentic Delivery?

Agentic Delivery is a systematic approach to software development where the **main agent orchestrates** and **subagents execute**. Instead of jumping straight into code, it guides your AI assistant through a structured workflow:

1. **Intent Recognition** — Classify the task (large feature / small change / bug fix)
2. **Doc Scanning** — Generate project context
3. **Requirement Analysis & Design** — Brainstorm approaches, get user approval
4. **Implementation Planning** — Break down into bite-sized tasks with dependencies
5. **Code Implementation** — Dispatch subagents per task, review between tasks
6. **Review Loop** — Spec compliance + code quality checks
7. **Summary** — Delivery report with artifacts

This isn't just a tool—it's a **delivery methodology** embodied in 12 composable skills.

Behavioral details live in the individual skills and `runtime-policies.md`. This README is an overview, not the source of truth for execution rules.

## Core Principles

### 1. Main Agent Orchestrates, Subagents Execute
- Main agent stays focused on orchestration, never loses context
- Each subagent gets **minimum context** for its specific task
- Stateless subagents: fresh instance per dispatch, no context pollution

### 2. Document-Driven Development
- Every stage produces persistent docs in `docs/<project>/<feature>/`
- Cross-session reuse: Skip stages if artifacts already exist
- Knowledge accumulates across sessions

### 3. Review Quality Gates
- Two-stage review: Spec Compliance → Code Quality
- Tiered fix strategy: Minor/Important (patch) vs Critical (reimplement)
- Max 3 review rounds, then escalate to user

## Features

- **3 Workflow Paths**: Large Feature (full pipeline) / Small Change (fast path) / Bug Fix (systematic debugging)
- **Parallel Task Execution**: Independent tasks run in parallel, file conflicts prevented at plan stage
- **Model Selection Strategy**: Use cheapest model per role (Fast/Standard/Capable)
- **Platform Adaptation**: Works on Claude Code, Codex, Cursor, OpenCode, Gemini
- **Cross-Session Recovery**: Resume from existing design-spec.md or implementation-plan.md

## Skills Included

### Core Orchestration
- **agentic-delivery** — Main entry point, 7-stage orchestration

### Custom Skills
- **doc-scanner** — Project structure extraction
- **doc-syncer** — Changelog update automation
- **review-fix-strategy** — Tiered fix decision framework

### From Superpowers
- **brainstorming** — Requirement analysis & design
- **writing-plans** — Implementation plan authoring
- **subagent-driven-development** — Task-by-task execution with review
- **dispatching-parallel-agents** — Parallel task orchestration
- **requesting-code-review** — Code review template
- **systematic-debugging** — 4-phase debugging methodology
- **verification-before-completion** — Evidence-before-claims principle
- **finishing-a-development-branch** — Branch completion workflow

**Total: 12 skills, 26 files**

## Installation

### Claude Code

Option 1: Via Plugin Marketplace (if available)
```bash
/plugin install agentic-delivery
```

Option 2: From GitHub
```bash
/plugin install https://github.com/xhyqaq/agentic-delivery
```

### Codex

```bash
# 1. Clone the repository
git clone https://github.com/xhyqaq/agentic-delivery.git ~/.codex/agentic-delivery

# 2. Create skills symlink
mkdir -p ~/.agents/skills
ln -s ~/.codex/agentic-delivery/skills ~/.agents/skills/agentic-delivery

# 3. Restart Codex
```

**Detailed instructions:** [.codex/INSTALL.md](.codex/INSTALL.md)

### Cursor / OpenCode / Gemini

See [docs/installation.md](docs/installation.md) for platform-specific guides.

## Quick Start

1. **Start a new feature:**
   ```
   User: I want to add user authentication with JWT tokens
   ```

2. **Agentic-delivery activates automatically**, based on description matching:
   ```
   I'm using the agentic-delivery skill to handle this task.

   Intent: Large Feature (multi-module, needs design)
   → Following Full Pipeline (Stage 1-7)

   Let me first scan your project...
   ```

3. **Follow the guided workflow:**
   - Answer clarifying questions
   - Review design in digestible sections
   - Approve implementation plan
   - Watch subagents work autonomously
   - Review task by task with implementation and fix checkpoints

## Example Workflows

### Large Feature
```
User request → Intent: Large Feature
  → Doc Scan → Brainstorming → Write Plan
  → Dispatch Implementers (parallel where possible)
  → Two-stage review per task
  → Mark task complete → Doc Sync
  → Summary report
```

### Small Change
```
User request → Intent: Small Change
  → Implement directly (main agent or subagent)
  → Code Quality Review only
  → Brief summary
```

### Bug Fix
```
User request → Intent: Bug Fix
  → Systematic Debugging (4 phases)
  → Code Quality Review
  → Root cause explanation
```

## Documentation

- **[Design Document](docs/design/agentic-delivery-design.md)** — Complete system design (11 chapters, 742 lines)
- **[Installation Guide](docs/installation.md)** — Platform-specific setup
- **[Quick Start Tutorial](docs/quick-start.md)** — First-time walkthrough

## External Dependencies (Not Included)

Some superpowers skills reference external skills that are **not included** in this package:

- **test-driven-development** — TDD workflow (referenced by systematic-debugging)
- **executing-plans** — Alternative to subagent-driven-development (referenced by writing-plans)
- **using-git-worktrees** — Git worktree isolation (referenced by subagent-driven-development)

If needed, install separately from [obra/superpowers](https://github.com/obra/superpowers).

## Architecture Highlights

- **Minimum Context Principle**: Each subagent receives only task text + relevant spec fragment, never full session history
- **File Conflict Prevention**: Plan stage marks file overlaps, prevents parallel execution of conflicting tasks
- **Stateless Subagents**: Fresh instance per dispatch via Claude Code's Task tool, no context carryover
- **Document Standardization**: Fixed templates embedded in subagent prompts (project-context.md, design-spec.md, implementation-plan.md, changelog.md)
- **Authority Model**: `runtime-policies.md` and the skills define behavior; design docs and README explain it

## License

MIT License - see [LICENSE](LICENSE)

## Contributing

Issues and pull requests welcome at [xhyqaq/agentic-delivery](https://github.com/xhyqaq/agentic-delivery).

## Credits

- **Superpowers Framework**: [obra/superpowers](https://github.com/obra/superpowers) — 8 of 12 skills adapted from this excellent workflow system
- **Design & Implementation**: xhy

---

**Built for autonomous agents. Made for human productivity.**
