---
name: linear
description: Use when working with Linear issues in the screenpipe monorepo — browse the backlog, triage, plan, propose improvements, or implement issues. Orchestrates triage/plan/propose/implement subagents for context-efficient processing. Triggers on "Linear", "issue", "backlog", "ticket", "BLU-", or any Linear team identifier.
---

# Linear Issue Workflow (screenpipe)

## Overview

Orchestrate Linear issue workflows: browse the backlog, pick issues, create plans, and execute them — keeping main context lean via subagents.

## Prerequisites

Either:
- **API key (default)**: `export LINEAR_API_KEY="lin_api_..."` (Linear Settings → API → Create Key).
- **MCP (alternative)**: configure the official Linear MCP server at `https://mcp.linear.app`. If MCP is active, prefer `mcp__linear__*` tools over the script — they handle auth via OAuth and avoid local key storage.

If neither is set, stop and tell the user which to configure.

## API Script

All non-MCP Linear calls go through:

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>
```

The `$(git rev-parse --show-toplevel)` ensures the script resolves correctly from any cwd (including worktrees).

Commands: `list-teams`, `list-projects`, `list-states`, `list-issues`, `get-issue`, `search-issues`, `create-issue`, `update-issue`, `update-status`, `add-comment`. Run with no args for full flag reference.

## screenpipe Context (read these BEFORE planning or implementing)

- `VISION.md` — product vision. Stability over features. Activation over new capabilities. No feature creep.
- `DESIGN.md` — UX/visual conventions.
- `TESTING.md` — regression checklist; mandatory before touching window mgmt, tray/dock, monitors, audio, or Apple Intelligence.
- `CLAUDE.md` — package managers (`bun` for JS/TS, `cargo` for Rust), file-header rule, multi-agent git etiquette ("never delete local code or use git reset").

When a Linear issue touches any TESTING.md-flagged area, the plan MUST cite the regression cases it intends to cover.

## Workflow

### Step 1: Determine Scope

Ask the user (or infer from context):
- **Browse backlog** → spawn `linear-triage` subagent
- **Specific issues** ("BLU-42, BLU-57") → skip to Step 2
- **Propose improvements** → spawn `linear-propose` subagent (Step 1b)

### Step 1b: Propose

Use the Task tool:

```
subagent_type: general-purpose
prompt: |
  You are a Linear proposal agent for the screenpipe monorepo. Read and follow:
    $(git rev-parse --show-toplevel)/.claude/skills/linear/linear-propose/SKILL.md

  Team key: <TEAM_KEY>
  Focus area: <user-specified or "broad sweep">
  Project root: <git rev-parse --show-toplevel>

  Linear API script:
    bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>

  Before proposing anything, read VISION.md — propose nothing that violates "stability over features".
```

Present returned proposals as a table, ask which to create, then create approved ones via `create-issue`.

### Step 2: Plan

For each selected issue, spawn `linear-plan`:

```
subagent_type: general-purpose
prompt: |
  You are a Linear planning agent. Read and follow:
    $(git rev-parse --show-toplevel)/.claude/skills/linear/linear-plan/SKILL.md

  Issue: <IDENTIFIER>
  Project root: <git rev-parse --show-toplevel>

  Linear API script:
    bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>

  Read VISION.md, DESIGN.md, TESTING.md before producing the plan.
```

**Parallelism**: independent issues → parallel; overlapping (same files) → serial. Ask if unsure.

### Step 3: Review Plans

Present each plan summary. Wait for explicit user approval.

### Step 4: Implement

For each approved plan, spawn `linear-implement`:

```
subagent_type: general-purpose
prompt: |
  You are a Linear implementation agent. Read and follow:
    $(git rev-parse --show-toplevel)/.claude/skills/linear/linear-implement/SKILL.md

  Issue: <IDENTIFIER>
  Plan:
  <paste full plan>

  Project root: <git rev-parse --show-toplevel>
  Main repo root: <absolute path>

  Linear API script:
    bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>

  MANDATORY: pull latest from main and work in a git worktree under ~/.screenpipe-worktrees/<IDENTIFIER>.
  NEVER touch existing worktrees in the repo (other agents may be using them — see CLAUDE.md).
```

### Step 5: Summary

Summarize implemented work, list follow-up issues created, surface blockers.

## Context Strategy

The orchestrator (you) holds minimal context:
- Issue identifiers + one-line summaries only
- Delegate all codebase exploration to subagents
- Delegate all code writing
- After Step 2: hold plan *summaries*, not full plans

## Important

- **Auto-set "In Progress"** when an issue is selected for planning or implementation. No confirmation needed.
- **Never auto-close** an issue. The user decides when to mark Done.
- **Announce write operations** before executing (create issue, update status, add comment).
- **Multi-agent etiquette**: another agent may already be working on an "In Progress" issue. Surface those separately and do not suggest them as new work.
- **screenpipe stability rule**: if a proposed change risks regressing a TESTING.md-flagged area, raise it loudly before proceeding.
