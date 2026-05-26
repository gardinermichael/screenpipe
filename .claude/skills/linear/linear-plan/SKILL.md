---
name: linear-plan
description: Use when planning a specific Linear issue — fetch the issue, explore the codebase, and produce a structured implementation plan with file paths, steps, tests, and risks. Triggers on "plan BLU-", "plan this issue", "make a plan for <issue ID>". Updates the Linear issue to "In Progress" and posts a planning comment.
---

# Linear Plan

## Overview

Given a Linear issue identifier, fetch its details, explore the screenpipe codebase, and create a detailed implementation plan. Update the issue status and post a planning comment.

## API Script

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>
```

## Required Reading (screenpipe-specific)

Before producing any plan, read:
- `VISION.md` — verify the issue aligns with "stability over features"
- `DESIGN.md` — for any UI/UX touchpoint
- `TESTING.md` — if the issue touches window mgmt, tray/dock, monitors, audio, or Apple Intelligence, the plan MUST list applicable regression cases
- `CLAUDE.md` — package manager rules (`bun` for JS/TS, `cargo` for Rust), file-header requirement

## Process

### 1. Fetch Issue Details

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" get-issue <IDENTIFIER>
```

Parse: title, description, comments, relations, parent/children, priority.

### 2. Explore the Codebase

- **Glob** to find relevant files
- **Grep** for related code/functions/components
- **Read** to understand existing patterns
- Identify which files change, how
- Check existing tests in the area

### 3. Create Plan

```markdown
## Plan for <IDENTIFIER>: <Title>

**Summary**: <1-2 sentences>

**screenpipe alignment**:
- Vision check: <how this aligns with stability/activation goals; or "N/A — pure bugfix">
- Regression risk areas (TESTING.md): <list, or "none">

**Files to modify**:
- `path/to/file.ts` — <what & why>

**New files** (if any):
- `path/to/new-file.ts` — <purpose>
- Remember: every new .rs/.ts/.tsx/.js/.jsx/.swift/.py file needs the screenpipe header (see CLAUDE.md)

**Implementation steps**:
1. <Specific actionable step with file paths>
2. ...

**Tests**:
- Unit/integration: <what to write/update>
- Regression (from TESTING.md): <specific cases to verify>

**Risks / Notes**:
- <Edge cases, dependencies, ordering concerns>
```

### 4. Update Linear

Immediately set the issue to "In Progress" (no confirmation):

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" update-status <IDENTIFIER> "In Progress"
```

Add a planning comment:

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" add-comment <IDENTIFIER> "Implementation plan created by Claude Code. Summary: <1-2 sentences>."
```

### 5. Return Plan

Return the full plan text to the orchestrator.

## Key Principles

- **Be specific**: exact file paths, function names, line references.
- **Be minimal**: the smallest change that solves the issue (screenpipe's stability rule).
- **Be honest**: flag unclear requirements or confusing code.
- **Follow existing patterns**: match project conventions; don't introduce new ones.
- **Always include tests**: unit + regression where TESTING.md applies.
