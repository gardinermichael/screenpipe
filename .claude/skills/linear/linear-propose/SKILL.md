---
name: linear-propose
description: Use when the user wants to surface improvements, bugs, tech debt, or features in the screenpipe codebase as new Linear issues. Triggers on "propose Linear issues", "what should we file", "audit for tech debt and file tickets", "find improvements". Returns categorized proposals for user approval; only creates issues after the user picks.
---

# Linear Propose

## Overview

Explore the screenpipe codebase to identify improvements, missing features, bugs, and tech debt. Present categorized proposals to the user. Create approved proposals as Linear issues.

## API Script

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" <command>
```

## screenpipe Filter (CRITICAL)

Before proposing anything, read `VISION.md`. Filter out:
- New features that aren't activation-critical
- "Nice to have" refactors with no stability benefit
- Anything that adds surface area without removing failure modes

screenpipe is **stability over features**. Propose stabilization, bug fixes, and activation improvements before novel features.

## Process

### 1. Understand Scope

User-specified focus (perf / UX / security / specific dir) or broad sweep.

### 2. Explore Systematically

- **Structure**: `package.json`, `Cargo.toml`, layout, configs
- **Entry points**: pages, routes, API endpoints, Tauri commands
- **Components & hooks**: reusability gaps, missing abstractions
- **Error handling**: missing try/catch, unhandled promise rejections, panics
- **Test coverage**: untested critical paths, missing regression cases per TESTING.md
- **Performance**: heavy renders, missing memo, N+1, large bundles, hot loops
- **Security**: exposed secrets, missing validation, injection vectors, IPC gaps
- **Accessibility**: ARIA, keyboard nav, screen reader
- **UX**: loading/error/empty states, optimistic updates
- **Code quality**: duplication, dead code, outdated deps, inconsistent patterns

Use Glob/Grep/Read. Be thorough but focused.

### 3. Cross-Reference Existing Issues

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" list-issues --team <KEY> --limit 50
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" search-issues "<keyword>"
```

Skip duplicates.

### 4. Categorize

| Category | Label | Description |
|----------|-------|-------------|
| Bug | Bug | Something broken |
| Performance | Performance | Optimization |
| Feature | Feature | Missing functionality (only if activation-critical per VISION.md) |
| Tech Debt | Tech Debt | Code quality, duplication |
| Security | Security | Vulnerabilities or hardening |
| Accessibility | Accessibility | A11y |
| UX | UX | UX gaps |
| Regression Risk | TESTING | Areas flagged in TESTING.md that lack regression coverage |

For each proposal:

```markdown
### <Title> [<Category>]

**Priority**: 1-Urgent / 2-High / 3-Medium / 4-Low
**Effort**: S / M / L
**Files**: `path/to/file.ts`, `path/to/other.ts`
**Vision alignment**: <stability / activation / both / "discuss with user">

**Problem**: <specific, with code locations>
**Suggestion**: <recommended fix>
```

### 5. Return Proposals

Return structured list to the orchestrator. The orchestrator presents and asks which to create.

## Key Principles

- **Be specific**: file paths, function names, line numbers.
- **Be actionable**: each proposal = one Linear issue.
- **Vision-aligned**: stability > features. If unsure, mark "discuss with user".
- **Avoid noise**: no trivial style nits, no subjective preferences.
- **Deduplicate**: always check existing Linear issues first.
