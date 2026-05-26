---
name: linear-implement
description: Use when executing a Linear issue's implementation plan — creates a git worktree, applies code changes, runs tests/lints, and posts results to Linear. Triggers on "implement BLU-", "execute the plan for <issue>", "code up <issue>". Files follow-up issues for any out-of-scope tech debt discovered.
---

# Linear Implement

## Overview

Execute a Linear issue's plan in an isolated git worktree, run screenpipe's checks, update Linear with results, and file follow-up issues for discoveries.

## API Script

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>
```

## Process

### 1. Pull Latest & Create Worktree

**Mandatory. Never work outside a worktree, and never reuse another agent's worktree (CLAUDE.md: "bunch of other agents working in parallel, never delete local code or use git reset").**

Place worktrees OUTSIDE the repo to avoid collision with other agents and the existing `worktrees/` dir:

```bash
MAIN_REPO=$(git rev-parse --show-toplevel)
WORKTREE_DIR="$HOME/.screenpipe-worktrees/<identifier-lowercase>"

cd "$MAIN_REPO"
git checkout main && git pull origin main
git worktree add "$WORKTREE_DIR" -b "<IDENTIFIER>"

# Symlink any local env files the project needs
[ -f "$MAIN_REPO/.env.local" ] && ln -sf "$MAIN_REPO/.env.local" "$WORKTREE_DIR/.env.local"

cd "$WORKTREE_DIR"
```

### 2. Understand the Plan

Read the provided plan. Identify files, order of operations, tests, the team key.

### 3. Use Superpowers Skills (if available)

If the environment has superpowers/related skills, invoke before proceeding:
- **brainstorming** — before creative work
- **tdd-workflow** / **test-driven-development** — before implementation
- **systematic-debugging** — when bugs surface
- **verification-before-completion** — before claiming done
- **writing-plans** — if the plan needs decomposition

If unavailable, continue normally.

### 4. Implement

- Edit/Write code per the plan
- **Add the screenpipe header to every new source file** (.rs/.ts/.tsx/.js/.jsx/.swift/.py) per CLAUDE.md
- Follow existing project patterns (`bun` for JS/TS, `cargo` for Rust)
- Keep changes minimal — scope discipline is mandatory
- Write tests for new functionality
- For TESTING.md-flagged areas (window mgmt, tray/dock, monitors, audio, Apple Intelligence) verify the listed regression cases

### 5. Run Checks

Check `package.json` / `Cargo.toml` for available scripts. Common screenpipe checks:

```bash
# JS/TS sides
bun run check          # if available
bun run lint
bun run typecheck
bun test

# Rust sides
cargo test
cargo clippy -- -D warnings
```

Fix the root cause of any failure. Re-run until green.

### 6. Handle Discoveries

For out-of-scope bugs/tech debt:

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" create-issue \
  --title "Found during <IDENTIFIER>: <brief>" \
  --team <TEAM_KEY> \
  --priority 3 \
  --description "Discovered while implementing <IDENTIFIER>. <Details>"
```

Track all follow-ups in your summary.

### 7. Update Linear

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" add-comment <IDENTIFIER> \
  "Implementation complete. Changes: <files & approach>. Tests: <what passes>. Regression checks: <from TESTING.md, if applicable>. Follow-ups: <issue IDs or 'none'>."
```

**Do NOT mark Done.** The user closes issues.

### 8. Return Summary

Report to orchestrator:
- Files changed, approach
- Tests written and passing
- All project checks passing (yes/no)
- Follow-up issue IDs
- Blockers (if any)

## Key Principles

- **Scope discipline**: only what the plan says. No "while I'm here" refactors.
- **Check before claiming done**: run lint, typecheck, tests, and regression cases.
- **File follow-ups, don't fix everything**: out-of-scope = new Linear issue.
- **Be transparent**: honest reporting beats green-washing.
- **Worktree hygiene**: never touch another agent's worktree, never `git reset`, never delete local code.
