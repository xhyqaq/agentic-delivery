# Installing agentic-delivery for Codex

Enable agentic-delivery skills in Codex via native skill discovery. Just clone and symlink.

## Prerequisites

- Git

## Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/xhyqaq/agentic-delivery.git ~/.codex/agentic-delivery
   ```

2. **Create the skills symlink:**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/agentic-delivery/skills ~/.agents/skills/agentic-delivery
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\agentic-delivery" "$env:USERPROFILE\.codex\agentic-delivery\skills"
   ```

3. **Restart Codex** (quit and relaunch the CLI) to discover the skills.

## Verify

```bash
ls -la ~/.agents/skills/agentic-delivery
```

You should see a symlink pointing to your agentic-delivery skills directory.

## Updating

```bash
cd ~/.codex/agentic-delivery && git pull
```

Skills update instantly through the symlink.

## Uninstalling

```bash
rm ~/.agents/skills/agentic-delivery
```

Optionally delete the clone: `rm -rf ~/.codex/agentic-delivery`.
