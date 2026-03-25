---
name: doc-scanner
description: "Use when starting a large feature and needing to understand project structure, conventions, and existing documentation before design work begins"
---

# Doc Scanner

Scan project structure and existing documentation to generate a standardized project context file. Uses **"Trust but Verify"** strategy: read external documentation as initial claims, verify critical assertions against code, and trust verified information while discarding contradictions.

## When to Use

- Called by `agentic-delivery` at Stage 2 (Doc Scan)
- When `docs/project-context.md` does not exist or is outdated
- When entering a new project for the first time

**Path Note:** File is always generated at `docs/project-context.md` in the current working directory, NOT in subdirectory projects.

## Core Philosophy: Trust but Verify

**Information Source Trust Hierarchy:**

1. **Code itself** - Ground truth (never lies)
2. **Our workflow-generated docs** - Trusted (recent + reviewed)
3. **External documentation** - Verify first, then decide:
   - ✅ Verified (match rate > 70%) → Trust
   - ⚠️ Partial (match rate 30-70%) → Flag discrepancies
   - ❌ Contradicted (match rate < 30%) → Discard, scan code

**Why verification saves resources:**
- 80% of projects have reasonably accurate docs → Save ~1700 tokens by trusting
- 20% have outdated docs → Catch errors via sampling (~300 tokens)
- Expected cost: 700 tokens vs 2000 tokens (full scan)

## The Process

### Phase 1: Pre-check (< 50ms)

**Check if external docs exist and are fresh:**

```bash
# 1. Check doc existence
if [ ! -f README.md ] && [ ! -f CLAUDE.md ] && [ ! -d docs/ ]; then
  # No docs → skip to Phase 3 (code-only scan)
fi

# 2. Check freshness (git log)
last_update=$(git log -1 --format="%ar" README.md)
if [[ $last_update == *"year"* ]] || [[ $last_update == *"years"* ]]; then
  # Docs > 1 year old → skip to Phase 3
fi
```

**Decision:**
- No docs OR docs > 1 year old → Skip to Phase 3 (code-only scan)
- Otherwise → Continue to Phase 2 (verify)

---

### Phase 2: Read & Verify (< 500ms)

#### Step 1: Read Documentation Claims

**Extract key claims from:**
- `README.md` - Tech stack, architecture overview
- `CLAUDE.md` / `AGENTS.md` - Coding conventions, project guidelines
- `docs/architecture.md` - Architecture patterns
- `CONTRIBUTING.md` - Development workflow

**Categories to extract:**
- **Tech Stack** (P0): Framework, language, package manager, test framework
- **Architecture** (P1): Layer structure, module organization
- **Conventions** (P2): Naming, import ordering, comment style

#### Step 2: Quick Verification

**Verification Rules:**

**P0 - Tech Stack (Must Verify):**
```bash
# Sample 10 random files from src/
sample_files=$(find src -type f -name "*.ts" -o -name "*.tsx" -o -name "*.js" | shuf | head -10)

# Check claimed framework
if doc says "React":
  matches=$(grep -l "from ['\"]react" $sample_files | wc -l)
  match_rate=$((matches * 100 / 10))

  if [ $match_rate -gt 70 ]; then
    echo "✅ React verified (${match_rate}%)"
  elif [ $match_rate -lt 30 ]; then
    echo "❌ React claim contradicted, check alternatives"
  else
    echo "⚠️ Partial React usage (${match_rate}%)"
  fi
```

**P1 - Architecture (Should Verify):**
```bash
# If docs claim "3-layer architecture"
# 1. Check directory structure
if [ -d src/ui ] && [ -d src/services ] && [ -d src/data ]; then
  echo "✅ Directory structure matches claim"

  # 2. Sample cross-layer imports
  violations=$(grep -r "import.*data" src/ui/ | wc -l)
  if [ $violations -eq 0 ]; then
    echo "✅ Architecture verified"
  else
    echo "⚠️ Architecture exists but not strictly enforced"
  fi
else
  echo "❌ Architecture claim contradicted"
fi
```

**P2 - Conventions (Optional Verify):**
```bash
# If docs claim "All functions must have JSDoc"
sample_functions=$(rg "^export (function|const)" src/ | shuf | head -10)
jsDoc_count=$(echo "$sample_functions" | grep -B1 "/\*\*" | wc -l)
match_rate=$((jsDoc_count * 100 / 10))

if [ $match_rate -lt 30 ]; then
  echo "❌ JSDoc claim contradicted (actual: ${match_rate}%)"
  echo "Inferring actual pattern from code..."
fi
```

#### Step 3: Decision Logic

For each claim:

```python
if match_rate > 70%:
    status = "✅ Verified"
    source = "Documentation (verified)"
    confidence = "High"
elif match_rate >= 30%:
    status = "⚠️ Partial"
    source = "Documentation + Code sampling"
    confidence = "Medium"
    note = f"Claimed in docs, actual usage {match_rate}%"
else:
    status = "❌ Contradicted"
    source = "Code sampling (doc discarded)"
    confidence = "High"
    action = "Discard claim, infer from code"
```

---

### Phase 3: Fallback - Code-Only Scan

**Triggered when:**
- No external documentation exists
- Documentation > 1 year old
- Critical claims (Tech Stack) contradicted (match rate < 30%)

**Actions:**

1. **Scan project structure**
   ```bash
   tree -L 3 -I "node_modules|build|dist|.git" > structure.txt
   find . -name "main.*" -o -name "index.*" -o -name "app.*"
   ```

2. **Infer tech stack from code**
   ```bash
   # Package manager
   ls -la | grep -E "lock\.yaml|lock\.json"

   # Framework (sample 20 files)
   rg "^import.*from ['\"]react" src/ | wc -l
   rg "^import.*from ['\"]vue" src/ | wc -l
   rg "^import.*from ['\"]@angular" src/ | wc -l

   # Test framework
   rg "from ['\"]vitest|from ['\"]jest|from ['\"]mocha" tests/
   ```

3. **Sample code conventions (20 random files)**
   ```bash
   # Naming patterns
   rg "^export (function|const) [a-z]" src/ | wc -l  # camelCase
   rg "^export (function|const) [A-Z]" src/ | wc -l  # PascalCase

   # Import ordering (manual inspection of 10 files)
   # Error handling patterns (search for try/catch, Result types)
   ```

4. **Generate project-context.md** (all from code)

---

### Phase 4: Generate project-context.md

**Output format depends on verification results:**

#### Format A: Verified Documentation (80% case)

```markdown
# Project Context

## Tech Stack (✅ Verified)

### React 18
**Source**: README.md claim
**Verification**: Sampled 10 files, 9/10 have React imports (90%)
**Status**: ✅ Verified
**Confidence**: High

### TypeScript
**Source**: package.json + file extensions
**Verification**: 96% of files are .ts/.tsx
**Status**: ✅ Verified

### Package Manager: pnpm
**Source**: pnpm-lock.yaml exists
**Commands**:
- Test: `pnpm test`
- Build: `pnpm build`
- Lint: `pnpm lint`

---

## Architecture (⚠️ Partially Verified)

### Three-Layer Pattern
**Documentation claim**: docs/architecture.md describes strict UI/Service/Data separation
**Verification**:
- Directory structure exists ✅
- Sampled 5 files, 3/5 follow pattern, 2/5 have cross-layer imports
**Actual situation**: Pattern exists but not strictly enforced
**Recommendation**: Follow pattern for new code, no forced refactoring of existing violations

---

## Coding Conventions

### Naming (✅ Verified)
**Documentation claim**: camelCase for functions, PascalCase for components
**Verification**: Sampled 20 exports, 19/20 match (95%)
**Status**: ✅ Verified

### JSDoc (❌ Documentation Ideal vs Reality)
**Documentation claim**: CLAUDE.md requires JSDoc for all functions
**Verification**: Sampled 10 functions, 2/10 have JSDoc (20%)
**Actual pattern**: TypeScript types serve as documentation, comments only when necessary
**Recommendation**: Follow actual pattern (types + essential comments)

---

## Directory Structure
```
src/
  components/  # React components
  hooks/       # Custom hooks
  utils/       # Utility functions
tests/
  unit/
  integration/
```

---

## External Documentation Status

- ✅ README.md (updated 3 months ago) - Tech stack accurate, state management outdated
- ✅ CLAUDE.md (updated 2 months ago) - Conventions idealized, reality differs
- ⚠️ docs/architecture.md (updated 1 year ago) - Pattern correct but not strictly followed
```

#### Format B: Code-Only (20% case)

```markdown
# Project Context

## Tech Stack (✅ Inferred from Code)

### React 18
**Source**: Code analysis (87 imports detected in src/)
**Confidence**: High
**Note**: No external documentation available or documentation contradicted

### TypeScript
**Source**: File extensions (.ts/.tsx: 96%)
**Confidence**: High

### Package Manager: pnpm
**Source**: pnpm-lock.yaml exists
**Commands**:
- Test: `pnpm test`
- Build: `pnpm build`

---

## Code Conventions (✅ Sampled from 20 files)

- **Naming**: camelCase for functions/variables, PascalCase for components (19/20 observed)
- **Imports**: React → third-party → local modules
- **Error Handling**: try-catch blocks + custom AppError class
- **Comments**: Minimal (5% of lines), TypeScript types as documentation

---

## Directory Structure
[tree output]

---

## External Documentation

⚠️ One of:
- No external documentation found
- Documentation > 1 year old (outdated)
- Documentation contradicted by code (discarded)

If you need domain knowledge or business rules, ask the user directly.
```

---

## Output Template Decision Tree

```
Has external docs? AND Fresh (< 1 year)?
├─ YES
│  ├─ Verify P0 (Tech Stack)
│  │  ├─ Match > 70% → Use Format A (✅ Verified sections)
│  │  ├─ Match 30-70% → Use Format A (⚠️ Partial sections)
│  │  └─ Match < 30% → Use Format B (Code-only)
│  └─ Verify P1 (Architecture) if claimed
│     └─ Flag discrepancies in Format A
│
└─ NO → Use Format B (Code-only)
```

---

## What NOT to Do

- ❌ Do NOT trust external docs blindly without verification
- ❌ Do NOT discard all docs without checking (wastes 80% of useful information)
- ❌ Do NOT read every file (sample 10-20 for verification)
- ❌ Do NOT include generated files, node_modules, build artifacts
- ❌ Do NOT infer conventions without statistical evidence (>50% observed)
- ❌ Do NOT include sensitive information (secrets, credentials, API keys)

## Key Principles

- **Code is ground truth** - When docs contradict code, code wins
- **Verify before trusting** - Sample 10-20 files to confirm critical claims
- **Flag uncertainties** - Better to say "⚠️ Partial" than pretend certainty
- **Cost-effective** - Verification saves 65% tokens vs full scan
- **Actionable output** - Mark each fact as ✅ Verified / ⚠️ Partial / ❌ Contradicted

## Lifecycle

**Created by:** `agentic-delivery` main agent at Stage 2
**Disposed:** Immediately after `project-context.md` is written
**Output persists:** As file on disk, reusable across sessions
**Verification cost:** ~300 tokens (vs 2000 for full scan)
