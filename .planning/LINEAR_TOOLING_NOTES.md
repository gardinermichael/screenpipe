# Linear Tooling Decision & Capabilities

## Decision

**Linear MCP (`mcp__plugin_linear_linear__*` tools) is the primary tooling path.** The wrapper script `bun .claude/skills/linear/scripts/linear-api.ts` serves as a fallback only. MCP was chosen because it natively supports projects, cycles, multi-label assignment, and dependency relations — surfaces the wrapper script lacks.

## MCP capabilities exercised in this session

- `list_teams` — fetch all teams in the Linear workspace
- `get_team` — retrieve team details by ID or key
- `save_issue` — create or update issues with:
  - Parent issue linking via `parentId`
  - Blocking dependencies via `blockedBy` and `blocks` arrays
  - Multi-label assignment via `labels` array
  - Priority, description, title, status
  - Returns new issue's ID, URL, and recommended `gitBranchName`

## MCP capabilities documented but not yet exercised

- `save_project` — create Linear projects to organize teams/sprints
- `list_cycles` — fetch cycles within a team
- `save_milestone` — create or update milestones
- `create_issue_label` — dynamically create labels
- `save_comment` — post comments on issues
- `save_document` — create wiki documents

## Wrapper-script gaps (fallback awareness)

If MCP becomes unavailable mid-execution and `bun linear-api.ts` is the only option, plan for these missing capabilities:

- No `create-project` — projects must be pre-created in Linear UI
- No `create-cycle` — cycles must be pre-created
- No `set-blocks` — dependency relations must be encoded in issue body text
- No `add-label` — labels must be pre-created in Linear or encoded in issue body

Workaround: store dependency + label intent in issue body until MCP resumes or script is extended.

## Auth mechanics

**MCP authentication:**
- Trigger: `mcp__plugin_linear_linear__authenticate`
- Flow: OAuth callback → `complete_authentication` tool
- State: OAuth state does **not persist** across sessions; expect to re-authenticate on every session resume
- Setup: user must approve one-time in the MCP settings before first use

**Wrapper script authentication:**
- Mechanism: `LINEAR_API_KEY` environment variable
- Setup: export `LINEAR_API_KEY=<api-key>` before running `bun linear-api.ts`
- Persistence: key persists for the shell session; survives across command invocations within one terminal

Preference: use MCP when available; fall back to wrapper script + env var if MCP auth fails.
