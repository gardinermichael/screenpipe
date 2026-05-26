# Upstream Implementation Issue Template

This template is copied verbatim into every new MIC-* implementation issue body. The base SHA must be refreshed at the time of issue creation: run `git fetch upstream && git rev-parse upstream/main` and replace the pinned SHA below if upstream has advanced since this document was last updated. Current pinned SHA: `dcfda38c0a0191c133d322a695ad05c96f483aa1` (verified 2026-05-26 against `refs/heads/main`).

---

## Goal
<one sentence: what the change achieves>

## Source
UPSTREAM_PROPOSAL.md §<section ID>

## Labels as text
```
tier: <0-6>
theme: <diagnosability|trust|privacy|local-ai|timeline|hygiene|rfc>
surface: <rust|tauri-ui|cli|docs>
suggested_worker: <haiku|sonnet|opus|codex|human-only>
upstream_pr: <yes|no>
repro_gated: <yes|no>
```

## Scope allowlist
Base branch: `upstream/main`
Base SHA: `dcfda38c0a0191c133d322a695ad05c96f483aa1` (refresh at creation per S0-6)

Allowed files/globs:
- <exact path or narrow glob>

Expected new files:
- <path or none>

## Dependencies
**Blocked by:**
- <issue ID or none>

**Blocks:**
- <issue ID or none>

*(Mirror the dependency list here for human readability AND set the corresponding `blockedBy`/`blocks` arrays via `mcp__plugin_linear_linear__save_issue`. The Linear edges are authoritative; this section is the human-readable echo.)*

## Acceptance criteria
- **Observable behavior:** <what users/maintainers will see/measure>
- **Tests/checks:** <unit/integration tests or manual verification steps>
- **Regression checklist:** <cite `TESTING.md` cases if touching audio, monitors, tray/dock, window management, or Apple Intelligence>
- **Pro-gate bypass diff-clean check:** passes (see `.planning/PRO_GATE_BYPASS_CHECKLIST.md`)

## Gates
1. **Design check** — proposed approach reviewed and approved
2. **Implementation in isolated worktree** — work at `~/.screenpipe-worktrees/<issue-id-lowercase>`
3. **Drift check** — tracked + untracked files compared to allowlist; emits `drift_report.json`
4. **Code review** — Claude reviewer plus Codex for nontrivial Rust/OAuth/diagnostic changes
5. **Upstream PR from upstream/main** — never from `ain`
