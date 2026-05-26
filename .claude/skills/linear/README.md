# Linear Skill (screenpipe-optimized fork)

A project-scoped Claude Code skill bundle for managing Linear issues in the screenpipe monorepo.

**Origin**: forked from [twanahc/claude-linear-skill](https://github.com/twanahc/claude-linear-skill), with security fixes and screenpipe-specific adaptations. Inspiration drawn from [wrsmith108/linear-claude-skill](https://github.com/wrsmith108/linear-claude-skill) (MCP integration patterns) and the official [Linear MCP server](https://mcp.linear.app).

## What's in this bundle

| Skill | Purpose |
|---|---|
| `linear` | Orchestrator. Routes to triage/plan/propose/implement. |
| `linear-triage` | Browse and filter the backlog; help the user pick issues. |
| `linear-plan` | Produce a detailed plan for one issue, with screenpipe regression-risk callouts. |
| `linear-propose` | Audit the codebase and propose new Linear issues, filtered through `VISION.md`. |
| `linear-implement` | Execute a plan in an isolated worktree, run checks, post results. |

## Auth

Pick one:

1. **Linear API key** (default for these scripts)
   ```bash
   export LINEAR_API_KEY="lin_api_..."
   ```
   Get the key at Linear → Settings → API → Create Key. Put `export ...` in `~/.zshrc` (user is on zsh).

2. **Linear MCP server** (preferred where available)
   - Configure `https://mcp.linear.app` in your Claude Code MCP config.
   - Uses OAuth, no local key storage.
   - When MCP is active, the subskills will prefer `mcp__linear__*` tools and skip the script. *(Manual switchover; the subskills currently call the script — convert to MCP when you wire it up.)*

## Changes from upstream (twanahc/claude-linear-skill)

### Security
- **Fixed GraphQL injection** in `listProjects` and `listIssues` — user-controlled flag values are now passed as GraphQL variables instead of being string-interpolated into the query. Original code allowed `team`/`status`/`assignee`/`label` values containing `"` to break out of the filter clause.
- **Bounded `--limit`** to `[1, 100]` to prevent unbounded queries.
- **Added MCP fallback note** in the auth-failure path.

### Portability
- **Removed `source ~/.bashrc &&`** prefix from all subskill commands (assumed bash; user runs zsh).
- **Replaced hardcoded `~/.claude/skills/linear/...` paths** with `$(git rev-parse --show-toplevel)/.claude/skills/linear/...` so the bundle works at project scope and survives worktrees.

### screenpipe fit
- **Worktree path moved outside the repo** (`~/.screenpipe-worktrees/<id>`) to avoid colliding with the monorepo's existing `worktrees/` directory and other agents.
- **Mandatory VISION.md / DESIGN.md / TESTING.md reads** baked into plan and propose subskills.
- **Regression checklist** required in plans touching TESTING.md-flagged areas (window mgmt, tray/dock, monitors, audio, Apple Intelligence).
- **screenpipe file-header rule** noted in `linear-implement`.
- **Multi-agent git etiquette** ("never delete local code or use git reset") propagated from `CLAUDE.md`.

### Triggering
- Descriptions rewritten with explicit "Use when..." + trigger phrases for better auto-discovery.

## Install path

This skill lives at `<monorepo-root>/.claude/skills/linear/`. Claude Code automatically discovers project-scoped skills in `.claude/skills/`.

## Smoke test

```bash
# Verify the script resolves
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts"   # prints usage

# With API key set:
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" list-teams
```

## License

Upstream is licensed under MIT (Twana Cheragwandi). This fork is also MIT.
