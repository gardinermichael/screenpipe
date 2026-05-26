# Codex Plan for Claude: Upstream Linear Sprint Breakdown

**Source doc:** `.planning/UPSTREAM_PROPOSAL.md`
**Target Linear team:** `MIC`
**Revision:** v3 (Codex APPROVE 2026-05-26 — see Changelog vs v1/v2)
**Status:** Ready for S0-1 backlog creation; S0-5 must resolve before dependent implementation starts.

## Summary

Use the project-scoped Claude Linear skills in `/Users/m/Repos/screenpipe/.claude/skills/linear/` to turn `.planning/UPSTREAM_PROPOSAL.md` into a Linear-ready execution backlog for team `MIC`.

Because the current `linear-api.ts` supports issue/project/status/comment operations but **not** project creation, cycles, dependency relations, or multi-label assignment, Claude should model this as a parent/child issue hierarchy. Dependencies, gates, labels, drift allowlists, and review rules live in issue bodies until either (a) `linear-api.ts` is extended with `create-project` / `create-cycle` / `set-blocks` / `add-label`, or (b) the Linear MCP server is configured (in which case `mcp__linear__*` tools handle projects/cycles/relations natively). See **S0-5** for the decision task.

Kiln is available as a user-scoped orchestration pipeline but is **not used** for this backlog — see [Kiln future-use note](#kiln-future-use-note-tied-to-s3-4) below.

## Linear Structure

Create one parent issue in `MIC`:

`Upstream contributions to screenpipe/screenpipe`

Create eight child "sprint" parent issues beneath it:

1. `Sprint 0: Linear scaffolding and safety gates`
2. `Sprint 1: Surgical upstream fixes`
3. `Sprint 2: Diagnosability core`
4. `Sprint 3a: OAuth and integration reliability`
5. `Sprint 3b: Privacy UX and memory destinations`
6. `Sprint 4: Local AI setup`
7. `Sprint 5: Timeline usability`
8. `Sprint 6: RFC-only heavy features`

Sprint 3 is split into 3a/3b mandatorily (per Codex v3 review) — OAuth ownership + privacy UX + memory connectors was too much for one sprint and the surfaces are independent enough to ship in parallel cycles.

Use titles like `[S1-1] Register missing browser automation Tauri commands`. Put all labels as text fields in the body, not actual Linear labels, unless **S0-5** confirms the tooling path supports multi-label updates.

Use this command family:

```bash
bun "$(git rev-parse --show-toplevel)/.claude/skills/linear/scripts/linear-api.ts" create-issue --team MIC --title "<title>" --parent <PARENT_ID> --description "<body>" --priority <1-4>
```

## Sprint Breakdown

### Sprint 0: Linear scaffolding and safety gates

Goal: make the backlog executable before implementation starts.

Tasks:
- `[S0-1] Create upstream parent/sprint issue hierarchy in MIC`
- `[S0-2] Add standard issue body template with drift tracking, gates, and source citation` — call out explicitly that labels and Blocks/Blocked-by edges live in body text because the current tooling cannot set Linear relations or multi-label assignments
- `[S0-3] Add upstream PR safety checklist for Pro-gate bypass exclusion`
- `[S0-4] Verify Linear tooling limits and document MCP/script path`
- `[S0-5] Decide tooling path` — Linear MCP (`mcp__linear__*` tools) is configured and is the chosen path for projects/cycles/relations/labels. Update the issue template to use real Linear edges. **Does not block S1 issue creation** — body-encoded deps are enough to file S1 issues, which are mostly independent. Resolve before any dependent-implementation start (i.e. before `linear-implement` fires on any issue whose `Blocked by` list is non-empty).
- `[S0-6] Refresh upstream/main base SHA at backlog-creation time` — run `git fetch upstream && git rev-parse upstream/main` and write the result into the issue-body template. Current pinned SHA: `dcfda38c0a0191c133d322a695ad05c96f483aa1` (verified 2026-05-26 against `refs/heads/main`).
- `[S0-7] File OOB GitHub issues against screenpipe/screenpipe upstream` — see **Out-of-band issues** section. Files **OOB-2 (F3 slow)** and **OOB-3 (F1 onnxruntime)** immediately. **OOB-1 (D1 Zoom)** is repro-gated and does not fire until **OOB-1a** captures a concrete symptom — do not file Zoom upstream without ground truth (per `UPSTREAM_PROPOSAL.md §D1`: "Repro the exact failure mode...before filing").

Acceptance:
- Every implementation issue body includes source section, base SHA, scope allowlist, dependencies, gates, reviewer requirement, and test plan.
- Tooling-path decision is recorded (MCP-first vs. script-wrappers) before any dependent implementation starts; not required to block S1 issue creation.
- Base SHA is recorded at creation time, not pinned from this doc.
- Issue template states that body-encoded dependencies/labels are the contract until tooling supports real Linear relations.

### Sprint 1: Surgical upstream fixes

Run these in parallel unless file overlap appears:

- `[S1-1] A1: Register missing browser automation Tauri commands`
  - Worker: Haiku or cheap Claude.
  - Gate: exact handler registration only.
- `[S1-2] C1: Flip obsidian daily summary default off`
  - Worker: Haiku.
  - Gate: one behavioral config change plus copy check.
- `[S1-3] A4: Add LM Studio CORS/restart warning copy`
  - Worker: Haiku.
  - Gate: copy-only UI change.
- `[S1-4] A3 v1 + E2: Add opener URL schemes, CI guard, and CONTRIBUTING note`
  - Worker: Codex or Sonnet.
  - Gate: TypeScript Compiler API test, no regex-only scanner.
  - **E2 explicit scope:** PR must also add a section to `CONTRIBUTING.md` documenting the Tauri 2.x compile-time URL-scheme allowlist constraint so the next contributor adding a deep-link button updates `main.json` in the same PR. Maps to `UPSTREAM_PROPOSAL.md §E2`.
- `[S1-5] F4: Add macOS CLI screen recording diagnostic logging`
  - Worker: Sonnet/Opus.
  - Gate: diagnostic-only, no permission behavior change.

Review:
- Per-PR review is enough.
- Do one batch drift review at sprint end (no two-model gate at S1 boundary — see Sprint Gates).
- Mandatory hard check: no Pro-gate bypass files leak into upstream PRs.

### Sprint 2: Diagnosability core

Tasks:
- `[S2-1] E1/H1: Add screenpipe doctor CLI scaffold and macOS doctor permissions`
- `[S2-2] H1-platform: Return structured not-implemented diagnostics on Windows/Linux`
- `[S2-3] F1: Add onnxruntime updater-restart panic/structured logging`
- `[S2-4] F2: Add doctor audio device enumeration and 100ms probe`
- `[S2-5] F3: Add screenpipe profile per-stage p50/p95/p99 output`
- `[S2-6] H6: Surface permission failure copy with real bundle/process identity`

Ordering:
- S2-1 blocks S2-2 and S2-4.
- **S2-6 blocked by S2-1** — H6 surfaces the real bundle/process identity output produced by the *macOS H1 implementation* that lives inside S2-1 (the umbrella CLI + macOS doctor permissions task). S2-2 is just the Windows/Linux not-implemented stubs and does not produce the data H6 needs. *(Corrects v2 which pointed S2-6 at S2-2.)*
- S2-3 and S2-5 can run in parallel (both standalone, no dependency on S2-1).
- Any audio-touching task must cite relevant `TESTING.md` audio regression cases.

### Sprint 3a: OAuth and integration reliability

Goal: close bug #3574 via OAuth ownership mode; ship `doctor integrations`; settle the Microsoft `prompt` repro+fix.

Tasks:
- `[S3a-1] A2a: Reproduce Microsoft prompt=consent+select_account against live tenant`
- `[S3a-2] A2a: If reproduced, switch Microsoft connectors to prompt=consent`
- `[S3a-3] B3 RFC: OAuth ownership mode design`
- `[S3a-4] B3 token migration spec: define behavior when switching managed <-> user-owned` (formerly S3-5 in v2; renumbered to reflect actual ordering)
- `[S3a-5] B3 implementation: Desktop-client PKCE/local exchange with SecretStore` (formerly S3-4 in v2; renumbered)
- `[S3a-6] H2: doctor integrations OAuth health and live refresh test`

Ordering (corrected from v2 per Codex review):
- `S3a-1 → S3a-2` (repro before fix).
- `S3a-3 → S3a-4 → S3a-5 → S3a-6` (strict chain).
  - **S3a-4 blocks S3a-5** because the migration spec defines the contract the implementation must honor (token reset / migrate / coexist semantics across modes).
  - **S3a-5 blocks S3a-6** because the diagnostic CLI reads runtime OAuth-mode state that only exists after the implementation lands.
  - **S3a-4 transitively blocks S3a-6** because doctor integrations classifies token state per mode — and what counts as "valid state" is defined in the migration spec, not the impl. A diagnostic shipped before the spec is finalized will misreport state during the very mode-switch flows users will hit. *(v2 incorrectly ran S3-5 in parallel with S3-6; Codex flagged this.)*
- Google prompt behavior is explicitly out of scope unless separately reproduced.

### Sprint 3b: Privacy UX and memory destinations

Goal: address the trust-UX surface UPSTREAM_PROPOSAL.md §G frames as "the single highest-leverage trust improvement upstream"; make memory destinations explicit.

Sprint 3b runs in parallel with 3a — none of its tasks depend on OAuth ownership landing.

Tasks:
- `[S3b-1] G1: Live data-flow inspector panel (read-only)` — what model processes data, what data leaves the machine, PII redaction state, credential location. Maps to `UPSTREAM_PROPOSAL.md §G1`.
- `[S3b-2] G2: First-call "what gets sent" provider dialog (one-time per provider+account)` — maps to `UPSTREAM_PROPOSAL.md §G2`.
- `[S3b-3] G3: PII-filter audit log (opt-in, metadata-only) at ~/.screenpipe/pii-audit.log` — maps to `UPSTREAM_PROPOSAL.md §G3`.
- `[S3b-4] H5 + C2: Memory destinations as explicit connectors` — Obsidian, Claude Code, Codex, filesystem-markdown, all **off by default** for new installs. Folds in C2's first-run opt-in dialog. Maps to `UPSTREAM_PROPOSAL.md §H5` and `§C2`. Touches `crates/screenpipe-core/src/memories/external_sync.rs:81-93`.

Ordering:
- All four S3b tasks run in parallel. No internal dependencies.
- 3b can open and close independently of 3a.

### Sprint 4: Local AI setup

Tasks:
- `[S4-1] B1: Add lmstudio provider type and preset storage compatibility`
- `[S4-2] B1: Add LM Studio provider card with inference and management URLs`
- `[S4-3] B1: List LM Studio models from GET /api/v1/models`
- `[S4-4] B1: Explicit model preload through POST /api/v1/models/load`
- `[S4-5] B1: Implicit load progress through streaming POST /api/v1/chat`
- `[S4-6] B1: Load-state retry and user-approved fallback`
- `[S4-7] B2: Add per-task model routing resolver and advanced UI`

Ordering (corrected from v1):
- `S4-1` → `S4-2` → `S4-3` → `{S4-4, S4-5}` → `S4-6` (strict chain).
- `S4-7` runs in parallel after `S4-1` (types are the only hard dep), but should not merge before `S4-2` patterns are settled.
- *(Corrects v1 which omitted S4-2 as a blocker for S4-3 — the provider card UI shell must exist before model listing renders into it.)*

### Sprint 5: Timeline usability

Tasks:
- `[S5-1] B4: Daily pipe-run rollup by pipe and date`
- `[S5-2] B4 config: Focus Assistant aggregate_history=daily`
- `[S5-3] B5 RFC: CandidateEvent schema and persistence choice`
- `[S5-4] B5: Implement local-first CandidateEvent persistence`
- `[S5-5] B5: Add validation UI writing to CandidateEvent store`
- `[S5-6] H4: Add daily activity review inbox`
- `[S5-7] B8: Add word/topic clustering to produce CandidateEvent rows`

Decisions:
- CandidateEvent v1 should be local-first Rust/SQLite exposed through Tauri commands.
- Use a constrained source enum plus metadata, not TS-only state.
- S5-3 blocks all CandidateEvent implementation.

### Sprint 6: RFC-only heavy features

File these as RFC issues only. Implementation starts only after explicit upstream maintainer approval in the RFC/discussion.

Tasks:
- `[S6-1] H3 RFC: Archive vs Action UI split`
- `[S6-2] B6 RFC: Multi-agent verification rounds`
- `[S6-3] B7 RFC: Export prompt for ChatGPT/Claude and re-ingest`
- `[S6-4] C3 RFC: Local wiki/memory explorer`
- `[S6-5] F5 RFC: Auto-updater pause/resume active recording`
- `[S6-6] A3 v2 RFC: IntegrationDef required_url_schemes and build.rs codegen`
- `[S6-7] E3 RFC: Centralize is_pro policy`

## Out-of-band issues (file as GitHub issues against screenpipe/screenpipe, not PRs)

These are surfaced by `UPSTREAM_PROPOSAL.md §"Issues to file"`. File OOB-2 and OOB-3 during Sprint 0 (via S0-7) so maintainer signal accumulates while Sprint 1 PRs are in flight. Each is zero-PR-risk and references a future diagnostic from this backlog. **OOB-1 (Zoom) is repro-gated** — do not file until OOB-1a produces ground truth.

- `[OOB-1a] D1 Zoom repro task` (Sprint 0 local task, not a GitHub issue) — reproduce Zoom failure mode in a real session: capture which call fails (token expiry? webhook? meeting fetch? scope?), HTTP exchange where possible, exact error response. Output goes into OOB-1's body when it eventually fires.
- `[OOB-1] D1: Zoom integration failures` — **blocked by OOB-1a.** Files only after repro produces a concrete symptom. Link `doctor integrations` (S3a-6) as the future diagnostic path. Maps to `UPSTREAM_PROPOSAL.md §D1`.
- `[OOB-2] F3: "Extremely slow on powerful hardware"` — file immediately. Point at `screenpipe profile` (S2-5) as the diagnostic path before anyone proposes optimizations. Maps to `UPSTREAM_PROPOSAL.md §F3`.
- `[OOB-3] F1: onnxruntime updater crash` — file immediately. Reference S2-3 logging plan; ask maintainers for stack traces. Maps to `UPSTREAM_PROPOSAL.md §F1`.

D2 (Google blocked) is intentionally **not** filed separately — B3 (S3a-5) is the proper fix and the PR body will reference issue `#3574`.

## Per-Issue Template

Every implementation issue body should use this exact structure:

```markdown
## Goal
<one sentence>

## Source
UPSTREAM_PROPOSAL.md §<A1/B3/etc>

## Labels as text
tier: <0-6>
theme: <diagnosability|trust|privacy|local-ai|timeline|hygiene|rfc>
surface: <rust|tauri-ui|cli|docs>
suggested_worker: <haiku|sonnet|opus|codex|human-only>
upstream_pr: <yes|no>
repro_gated: <yes|no>

## Scope allowlist
Base branch: upstream/main
Base SHA: <set at backlog-creation time per S0-6; do not hardcode>
Allowed files/globs:
- <exact path or narrow glob>
Expected new files:
- <path or none>

## Dependencies
Blocked by:
Blocks:

(Body-encoded until S0-5 resolves whether real Linear edges are available
via MCP or via extended linear-api.ts wrappers. Treat body text as the
authoritative dependency contract until then.)

## Acceptance criteria
- Observable behavior:
- Tests/checks:
- Regression checklist:
- Pro-gate bypass diff-clean check passes.

## Gates
1. Design check
2. Implementation in isolated worktree under ~/.screenpipe-worktrees/<issue>
3. Drift check: tracked + untracked files compared to allowlist; emits
   drift_report.json (see Drift and Review Rules)
4. Code review: Claude reviewer plus Codex for nontrivial Rust/OAuth/diagnostic changes
5. Upstream PR from upstream/main, never from ain
```

## Drift and Review Rules

Claude should bake these into every issue and sprint summary:

- Work starts from `upstream/main`, not `ain`.
- Worktree path: `~/.screenpipe-worktrees/<identifier-lowercase>`.
- Drift check must include:
  - `git diff --name-only <base-sha>..HEAD`
  - `git ls-files --others --exclude-standard`
  - comparison against allowed files/globs.
- **Drift artifact:** the drift check emits `drift_report.json` (kept in the worktree, never pushed) with shape:

  ```json
  {
    "issue": "MIC-<num>",
    "base_sha": "<sha>",
    "head_sha": "<sha>",
    "expected_globs": ["..."],
    "unexpected_files": ["..."],
    "missing_expected": ["..."],
    "drift_approved": []
  }
  ```

  Gate 3 blocks if `unexpected_files` is non-empty unless each entry is acknowledged by an issue comment `drift-approved: <path> — <reason>`, which the reviewer then mirrors into `drift_approved[]` before passing.
- Every PR must be clean of local Pro-gate bypass edits: `is_pro` flips, `cloud_subscribed` transforms/injections, per-card `isPro = true` overrides.
- Use `bun` for JS/TS and `cargo` for Rust.
- New source files must include the screenpipe header.
- If a task touches audio, monitors, tray/dock, window management, or Apple Intelligence, the Linear plan must cite matching `TESTING.md` cases.

### Sprint Gates (two-model review)

A sprint cannot close until:

- Every issue passes all 5 per-issue gates **or** is explicitly moved to a later sprint with rationale in a comment.
- A sprint-level drift report — the union of every issue's `drift_report.json` files-touched set — is filed as a comment on the sprint parent issue.
- **Two-model review for Sprint 2 onward.** One Claude pass (this session or `code-reviewer` agent) plus one Codex pass (`/codex:rescue` or `mcp__Multi-CLI__Ask-Codex` with `model: "gpt-5.3-codex"`) review the sprint as a whole. Both must approve before the next sprint opens. **Skipped at Sprint 1** because Sprint 1 is 5 trivial PRs and the overhead outweighs the benefit; restart at Sprint 2.

## Claude Skill Routing

Use the project Linear skills this way:

- Use `linear-propose` only to create or refine the backlog from this plan.
- Use `linear-plan` per issue after the issue exists; it should post the concrete file-level implementation plan.
- Use `linear-implement` only after the issue plan is approved.
- Kiln is **not** used by default for this backlog — see note below.

Assumption: `MIC` is the intended Linear team and `LINEAR_API_KEY` or Linear MCP auth is already configured. If neither auth path is configured, Claude should stop before creating issues and report the missing auth setup.

## Kiln future-use note (tied to S3a-5)

Kiln is intentionally **not** used to orchestrate this backlog. The upstream contribution surface is the wrong shape for Kiln — each issue produces a small, isolated PR against an external repo, started from a clean `upstream/main` worktree. Kiln's value (persistent minds accumulating codebase knowledge across milestones) does not pay off when every chunk starts cold.

**One exception worth flagging for the future:** `S3a-5` (B3 OAuth ownership mode implementation; was `S3-4` in v2). If during scoping that issue grows past a single ~2-3 day PR — multi-file Rust changes in `OAuthConfig`, parallel Tauri-UI work for the global "OAuth ownership" settings section, docs/CONTRIBUTING updates, and the token-migration path defined by S3a-4 — *then* it may make sense to spin up a Kiln milestone scoped to just that issue. Triggers for revisiting:

- Estimate during `linear-plan` for S3a-5 exceeds 3 days, OR
- Touch surface spans ≥ 3 of `{Rust crate, Tauri UI, docs, migrations}`, OR
- The token-migration spec (S3a-4) defines semantics that materially expand S3a-5's surface beyond a single PR.

Decision rule: revisit at the end of S3a-4 (migration spec sign-off — that's the last point where scope is still cheap to change). Do not pre-commit to Kiln before then.

---

## Changelog

### v3 — Codex review pass (2026-05-26)

Codex returned `CHANGES` on v2. Six edits applied:

- **Mandatory S3a/S3b split** (was Codex's #1 + #2 revise) — Sprint 3 was too large with the G-series and H5/C2 restored. Now `Sprint 3a: OAuth and integration reliability` and `Sprint 3b: Privacy UX and memory destinations` are two separate parent issues that run in parallel.
- **S3a OAuth chain rewired** (was Codex's #5 reject of v2's "S3-5 parallel with S3-6") — token migration spec now blocks the implementation, which blocks the diagnostic: `S3a-3 (RFC) → S3a-4 (migration spec) → S3a-5 (impl) → S3a-6 (doctor integrations)`. Reasoning: doctor integrations classifies token state per mode, and what counts as "valid state" is defined in the migration spec, so a diagnostic shipped before the spec is finalized would misreport during mode-switch flows.
- **S2-6 dependency fix** (was Codex's #6 reject of v2's "S2-6 blocked by S2-2") — S2-6 (H6 permission copy) is now blocked by **S2-1** (which contains the macOS H1 implementation that actually produces the bundle/process-identity data). S2-2 is just the Win/Linux not-implemented stubs and produces no data.
- **OOB-1 Zoom is repro-gated** (was Codex's #3 + #12 revise) — Zoom upstream issue does not fire until **OOB-1a** (Sprint 0 local repro task) captures a concrete symptom. F1 and F3 OOB issues still file immediately during S0-7.
- **S0-5 scope narrowed** (was Codex's #10 revise) — tooling-path decision no longer blocks S1 issue creation. Body-encoded deps are enough to file the mostly-independent S1 backlog. S0-5 only blocks dependent-implementation starts (any issue with a non-empty `Blocked by` list).
- **Kiln note retargeted** — references in the Kiln future-use note now point at `S3a-5` (the new identifier for the B3 implementation) and `S3a-4` (the migration spec) instead of v2's `S3-4` / `S3-5`.

Codex edits **kept as-is from v2**: #4 (E2 explicit in S1-4), #7 (Sprint 4 chain), #8 (drift_report.json artifact), #9 (two-model sprint review from Sprint 2 onward), #11 (SHA refresh at backlog-creation time).

### v2 — Claude review pass (2026-05-26)

v2 applied 12 edits to the original Codex-authored v1 draft. v1 dropped a significant slice of `UPSTREAM_PROPOSAL.md` (the G-series privacy UX cluster, H5 memory destinations, C2, OOB issues, E2) and had several dependency errors. v2 restored the dropped content and fixed dependencies, drift mechanics, and sprint-gate policy. See git history of this file for the v2 changelog the v3 pass replaced.

### v1 — Codex draft

Original Codex-authored sprint breakdown saved at `.planning/LINEAR_UPSTREAM_PLAN.md` (the prior archived draft retains its own history; v1 of the current file is reachable via git history before the v2 commit).
