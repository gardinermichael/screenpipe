---
name: linear-triage
description: Use when the user wants to browse, filter, or pick Linear issues to work on next. Triggers on "browse Linear", "what's in the backlog", "show me issues", "what should I work on", and similar discovery requests. Returns selected issues with full context for downstream planning.
---

# Linear Triage

## Overview

Browse the Linear backlog for the screenpipe monorepo, filter by team/status/label/assignee, and help the user select issues. Return selected issues with full details to the orchestrator.

## API Script

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>
```

## Process

### 1. Discover Teams

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" list-teams
```

Ask which team(s) to focus on.

### 2. Check What's Already In Progress

**Before browsing available work**, check what's being actively handled:

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" list-issues --team <KEY> --status "In Progress"
```

Present these separately:

```
Currently in progress (other agents may be handling these — do NOT suggest as new work):

| ID      | Title                        | Assignee   |
|---------|------------------------------|------------|
| BLU-38  | ...                          | —          |
```

### 3. Browse Available Issues

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" list-issues --team <KEY> --status "Todo"
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" list-issues --team <KEY> --status "Backlog"
```

User refinements:
- `--assignee "Name"`
- `--label "Bug"` / `"Feature"`
- `--limit 50`

Search:
```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" search-issues "auth redirect bug"
```

### 4. Present Issues

```
Available to work on:

| #  | ID      | Title                        | Priority | Labels     |
|----|---------|------------------------------|----------|------------|
| 1  | BLU-42  | Fix auth redirect loop       | Urgent   | Bug        |
| 2  | BLU-57  | Add dark mode toggle         | Medium   | Feature    |
```

### 5. Help Select

User picks by number, asks for detail, refines filters, or searches.

### 6. Fetch Full Details

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" get-issue <IDENTIFIER>
```

### 7. Return Results

Return selected issues with: identifier, title, full description, comments, related issues, priority, labels. The orchestrator uses this for planning.

## screenpipe Note

When showing issues that touch core capture/audio/monitor/UI features, flag them as **regression-sensitive** so the user knows TESTING.md applies.
