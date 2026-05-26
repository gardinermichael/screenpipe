# Linear Plan — Upstream Contribution Sprints

**Source doc:** `UPSTREAM_PROPOSAL.md`
**Target Linear team:** `MIC` (Mickies)
**Drafted:** 2026-05-26
**Status:** Draft for Codex review — **not yet pushed to Linear**

This is the proposed Linear breakdown of `UPSTREAM_PROPOSAL.md`. Reviewer (Codex): challenge sprint shape, dependency edges, drift-tracking fields, and checkpoint gates before any issue is created.

---

## 1. Linear hierarchy

```
Team:     MIC
Project:  "Upstream contributions to screenpipe/screenpipe"
  ├── Cycle 1 — Tier 1: Surgical fixes
  ├── Cycle 2 — Tier 2a: Diagnosability core
  ├── Cycle 3 — Tier 2b: Trust boundaries
  ├── Cycle 4 — Tier 3: Local AI setup
  ├── Cycle 5 — Tier 4: Timeline usability
  └── Cycle 6 — Tier 5: RFCs & heavy features
```

One Linear `Cycle` per sprint. Sprints run sequentially (gated by checkpoint review — see §5). Inside a sprint, independent issues run in parallel.

### Labels

| Label | Purpose |
|---|---|
| `tier-1` … `tier-5` | Maps back to roadmap tiers in source doc |
| `theme:diagnosability` / `theme:trust` / `theme:local-ai` / `theme:timeline` / `theme:hygiene` | Cross-cutting themes |
| `agent:haiku` / `agent:codex` / `agent:claude-opus` / `agent:human-only` | Suggested worker for the task |
| `surface:rust` / `surface:tauri-ui` / `surface:cli` / `surface:docs` | Codebase surface (drives reviewer routing) |
| `repro-gated` | Cannot be implemented until a live repro exists |
| `rfc-first` | Needs RFC issue + maintainer ack before implementation |
| `upstream-pr` | Output is a PR against `screenpipe/screenpipe:main` |
| `local-only` | Stays on this fork (e.g. drift-tracking scaffolding) |

---

## 2. Issue template (every implementation issue uses this body)

```markdown
## Goal
<one sentence — what is true after this lands?>

## Source
UPSTREAM_PROPOSAL.md §<section ID>

## Files in scope
- path/to/file.rs:LINE
- path/to/other.ts:LINE
(Listing the file set up front bounds the diff and lets the reviewer flag drift.)

## Drift-tracking baseline
- Base branch: `upstream/main`
- Base SHA at issue creation: `<filled at creation time>`
- Worktree path: `~/.screenpipe-worktrees/MIC-<num>`
- Touched-file allowlist: <same as "Files in scope">

If the implementing agent edits files outside the allowlist, the
checkpoint reviewer MUST flag it (drift) and either expand the allowlist
with justification or revert the out-of-scope edits.

## Acceptance criteria
- [ ] <observable behavior 1>
- [ ] <observable behavior 2>
- [ ] All tests in <list> pass
- [ ] `cargo test` (Rust changes) / `bun test` (TS changes) green
- [ ] No file outside "Files in scope" modified, OR drift documented in PR body
- [ ] PR diff is clean of Pro-gate bypass changes (see §6 hard rule)

## Checkpoint gates (must complete in order)
1. [ ] **Design check** — issue restated, ambiguity flushed, dependencies confirmed
2. [ ] **Implementation** — agent works in isolated worktree
3. [ ] **Drift check** — `git diff --stat` reviewed against allowlist
4. [ ] **Code review** — `code-reviewer` agent OR `mcp__Multi-CLI__Ask-Codex` (`gpt-5.3-codex`)
5. [ ] **Upstream PR** — branched off `upstream/main`, not `ain`

## Suggested worker
agent:<haiku|codex|claude-opus|human-only> — <one-line rationale>

## Dependencies
Blocks: MIC-<n>, …
Blocked by: MIC-<n>, …

## Repro / setup required
<any pre-work the implementer must do before coding>
```

---

## 3. Sprint definitions

Each sprint has: **goal**, **exit criteria**, **issues**, **dependency graph snippet**.

### Sprint 1 — Tier 1: Surgical fixes

**Goal:** Five small, near-zero-risk PRs landed upstream to establish contributor trust and prove the workflow.

**Exit criteria:** All five PRs filed against `screenpipe/screenpipe:main`. At least three merged or one round of maintainer review received on each.

| MIC# (planned) | Title | Source | Surface | Effort | Agent |
|---|---|---|---|---|---|
| S1-1 | Register four missing browsers-automation Tauri commands | A1 | rust | 5 lines | haiku |
| S1-2 | Flip `obsidian-daily-summary` `defaultOn` to `false` | C1 | tauri-ui | 1 line | haiku |
| S1-3 | Add LM Studio CORS/restart warning copy in existing panel | A4 | tauri-ui | 10 min | haiku |
| S1-4 | Allowlist missing URL schemes + add CI guard test (TS Compiler API) + CONTRIBUTING note | A3 v1 | tauri-ui + docs | 1 hr | claude-opus |
| S1-5 | Intel macOS CLI: log bundle ID + parent process + actual screencapture result + `tccutil` instructions | F4 (diagnostic only) | rust + cli | half day | claude-opus |

**Dependencies inside sprint:** none — all five run in parallel.

### Sprint 2 — Tier 2a: Diagnosability core

**Goal:** Turn the largest silent-failure classes into actionable structured errors. Establish the `screenpipe doctor` umbrella.

**Exit criteria:** `screenpipe doctor permissions` (macOS) ships; `doctor audio` and `profile` mode ship as separate PRs; `onnxruntime` updater path emits structured panic logs.

| MIC# | Title | Source | Surface | Depends on | Effort |
|---|---|---|---|---|---|
| S2-1 | `screenpipe doctor` umbrella CLI scaffold | E1 | rust + cli | — | 1 day |
| S2-2 | `doctor permissions` (macOS): bundle ID, parent chain, actual API call, remediation copy | H1 | rust + cli | S2-1, S1-5 | 1-2 days |
| S2-3 | `doctor permissions` (Win/Linux): structured "not implemented" diagnostic | H1 | rust + cli | S2-1 | half day |
| S2-4 | `onnxruntime` updater-restart panic hook + structured logging | F1 | rust | — | 1 day |
| S2-5 | `doctor audio` — device enum + 100ms probe per input | F2 | rust + cli | S2-1 | 1-2 days |
| S2-6 | `screenpipe profile` mode — per-stage p50/p95/p99 to stderr | F3 | rust | — | 1-2 days |
| S2-7 | Permission failure copy that names real bundle/process at point of failure | H6 | tauri-ui | S2-2 | half day per site |

**Parallelizable inside sprint:** S2-1, S2-4, S2-6 in parallel. S2-2/S2-3/S2-5 after S2-1. S2-7 after S2-2.

### Sprint 3 — Tier 2b: Trust boundaries

**Goal:** Close bug #3574 via OAuth ownership mode; ship the three privacy UX surfaces (data-flow inspector, first-call dialog, PII audit log); make memory destinations explicit.

**Exit criteria:** #3574 closed by PR; G1/G2/G3 merged or under maintainer review; memory destinations panel opt-in.

| MIC# | Title | Source | Surface | Depends on | Effort |
|---|---|---|---|---|---|
| S3-1 | Repro Microsoft `prompt=consent+select_account` against live tenant | A2a (repro) | rust | — | 1 hr |
| S3-2 | If reproed: switch Microsoft connectors to `prompt=consent` | A2a (fix) | rust | S3-1 | trivial |
| S3-3 | OAuth ownership mode: design RFC (storage, flow split, UI shape) | B3 (design) | docs | — | 1 day |
| S3-4 | OAuth ownership mode: implement Desktop-client PKCE + local exchange + 7-day testing warning | B3 (impl) | rust + tauri-ui | S3-3 | 2-3 days |
| S3-5 | `doctor integrations`: OAuth health + live refresh test per connector | H2 | rust + cli | **S3-4 (strict)** | 1-2 days |
| S3-6 | Live data-flow inspector panel (read-only) | G1 | tauri-ui | — | 1-2 days |
| S3-7 | First-call "what gets sent" provider dialog (one-time per provider) | G2 | tauri-ui | — | half day |
| S3-8 | PII-filter audit log (opt-in, metadata-only) | G3 | rust | — | half day |
| S3-9 | Memory destinations as explicit connectors (Obsidian/Claude/Codex/fs-md, all off by default) | H5 + C2 | rust + tauri-ui | — | 2-3 days |

**Strict ordering:** S3-1 → S3-2. S3-3 → S3-4 → S3-5. Everything else parallel.

### Sprint 4 — Tier 3: Local AI setup

**Goal:** First-class LM Studio integration (lifecycle-aware) + per-task model routing.

**Exit criteria:** `lmstudio` is a selectable provider type with model lifecycle; per-task routing usable from preset UI.

| MIC# | Title | Source | Surface | Depends on | Effort |
|---|---|---|---|---|---|
| S4-1 | Add `"lmstudio"` to `AIProviderType` + tauri types | B1 (types) | tauri-ui | — | 30 min |
| S4-2 | `LMStudioProviderCard` UI (split inference + management URLs) | B1 (UI) | tauri-ui | S4-1 | half day |
| S4-3 | Model listing via `GET /api/v1/models` with loaded/unloaded state | B1 (mgmt) | tauri-ui | S4-2 | half day |
| S4-4 | Explicit pre-load: `POST /api/v1/models/load` (non-streaming, indeterminate progress) | B1 (load-a) | tauri-ui | S4-3 | half day |
| S4-5 | Implicit load progress via `POST /api/v1/chat stream:true` events | B1 (load-b) | tauri-ui | S4-3 | half day |
| S4-6 | Load-state-aware inference retry + optional fallback model routing | B1 (resilience) | tauri-ui | S4-4, S4-5 | half day |
| S4-7 | Per-task model routing: `models.{chat,embedding,vision,summarization}` on `AIPreset` | B2 | tauri-ui | S4-1 | 1-2 days |
| S4-8 | RFC issue: per-task routing surface (if S4-7 surface change is large) | B2 (rfc) | docs | — | half day |

**RFC gate:** S4-8 must be filed and acked before S4-7 starts if the routing surface materially changes user-visible preset UI.

### Sprint 5 — Tier 4: Timeline usability

**Goal:** Kill pipe-run flood; introduce `CandidateEvent` model; ship the activity review inbox.

**Exit criteria:** Daily rollup live for Focus Assistant; `CandidateEvent` model lands; H4 inbox renders candidates; B8 clustering writes candidates.

| MIC# | Title | Source | Surface | Depends on | Effort |
|---|---|---|---|---|---|
| S5-1 | Daily pipe-run rollup: `pipe-run-day:<pipe>:YYYY-MM-DD` (opt-in, no migration) | B4 | tauri-ui | — | 1-2 days |
| S5-2 | Focus Assistant manifest: `aggregate_history: "daily"` | B4 (config) | docs/config | S5-1 | trivial |
| S5-3 | RFC: `CandidateEvent` schema (Rust crate vs TS-only, enum vs string `source`, required fields) | B5 (rfc) | docs | — | half day |
| S5-4 | Introduce `CandidateEvent` model + persistence layer | B5 (impl) | rust + tauri-ui | S5-3 | 1-2 days |
| S5-5 | Validation UI: ✓ / ✕ / ✎ inline buttons writing to `CandidateEvent` store | B5 (ui) | tauri-ui | S5-4 | 1 day |
| S5-6 | Daily activity review inbox surface (confirm/wrong/merge/ignore/export) | H4 | tauri-ui | **S5-4 (strict)**, S5-5 | 2-3 days |
| S5-7 | Word-cluster job → `CandidateEvent` rows (uses B1/B2 embedding model) | B8 | rust + tauri-ui | S5-4, S5-6, S4-7 | 2-3 days |

**Strict ordering:** S5-3 → S5-4 → S5-5 → S5-6 → S5-7. S5-1/S5-2 parallel to the chain.

### Sprint 6 — Tier 5: RFCs & heavy features

**Goal:** File RFC issues; only one (or none) graduates to implementation per cycle.

**Exit criteria:** Each item below is filed as a Linear issue with `rfc-first` label and an upstream Discussion/issue link. Implementation only if maintainer aligns.

| MIC# | Title | Source | Status | Effort if greenlit |
|---|---|---|---|---|
| S6-1 | RFC: "Archive vs Action" top-level UI split | H3 | RFC only | multi-PR |
| S6-2 | Multi-agent verification rounds (opt-in, cost-heavy) | B6 | RFC + impl | 1-2 days |
| S6-3 | "Export prompt for ChatGPT/Claude" hand-off + re-ingest | B7 | RFC + impl | half day |
| S6-4 | Local wiki/memory explorer (read-only v1) | C3 | RFC + impl | 3-5 days |
| S6-5 | Auto-updater graceful pause/resume of active recording | F5 | RFC + impl | unknown |
| S6-6 | `IntegrationDef::required_url_schemes` + `build.rs` codegen | A3 v2 | RFC + impl | half day |
| S6-7 | Centralize `is_pro` policy across connectors | E3 | RFC + impl | 1-2 days |

### Out-of-band issues (file, do not PR)

| MIC# | Title | Source |
|---|---|---|
| OOB-1 | Issue: Zoom integration failures — gather symptoms, link `doctor integrations` (S3-5) once shipped | D1 |
| OOB-2 | Issue: "Extremely slow on powerful hardware" — point at `screenpipe profile` (S2-6) for repro | F3 |
| OOB-3 | Issue: `onnxruntime` updater crash — reference S2-4 logging plan; ask maintainers for stack traces | F1 |

---

## 4. Cross-sprint dependency graph (text form)

```
Sprint 1 ─┬─ S1-5 ──────────────┐
          └─ (others standalone) │
                                 ▼
Sprint 2 ── S2-1 ──┬── S2-2 ──── S2-7
                   ├── S2-3
                   └── S2-5
Sprint 2 ── S2-4   (parallel)
Sprint 2 ── S2-6   (parallel)

Sprint 3 ── S3-1 ── S3-2          (Microsoft track)
Sprint 3 ── S3-3 ── S3-4 ── S3-5  (OAuth ownership track, strict)
Sprint 3 ── S3-6, S3-7, S3-8, S3-9 (parallel privacy track)

Sprint 4 ── S4-1 ── S4-2 ── S4-3 ─┬─ S4-4 ──┐
                                  └─ S4-5 ──┴── S4-6
Sprint 4 ── S4-1 ── S4-7  (per-task routing, parallel to lifecycle chain)

Sprint 5 ── S5-1 ── S5-2          (rollup track)
Sprint 5 ── S5-3 ── S5-4 ── S5-5 ── S5-6 ── S5-7   (candidate-event chain, strict)
        (S5-7 also depends on S4-7 for embedding model routing)

Sprint 6 ── RFC-gated; no auto-graduation.
```

---

## 5. Gated checkpoints & drift tracking

### 5.1 Per-issue gates (recap from template)
1. **Design check** — restate, list ambiguities, confirm deps.
2. **Implementation** — isolated worktree at `~/.screenpipe-worktrees/MIC-<num>`.
3. **Drift check** — `git diff --stat`, compare modified files vs allowlist.
4. **Code review** — `code-reviewer` agent OR Codex via `mcp__Multi-CLI__Ask-Codex` (`model: "gpt-5.3-codex"`).
5. **Upstream PR** — branched off `upstream/main`, never off `ain`.

### 5.2 Sprint gates
A sprint cannot close until:
- Every issue passes all 5 gates **or** is explicitly moved to a later cycle with rationale in a comment.
- The sprint's `Exit criteria` (above) are met.
- Drift report is filed as a comment on the sprint's project: "Files touched across all issues vs allowlist union." Any out-of-allowlist edits must be re-reviewed.
- Two-model review on the sprint as a whole: one Claude pass + one Codex pass via `/codex:rescue` or Multi-CLI MCP. Both must approve before next sprint opens.

### 5.3 Drift tracking mechanics
For each issue, on creation the orchestrator records:
- `git -C ~/Repos/screenpipe rev-parse upstream/main` → base SHA
- The full file allowlist (verbatim from "Files in scope")

On gate 3 (drift check), the orchestrator computes:
- `git diff --name-only <base-sha>..HEAD` (inside the worktree)
- Diff against allowlist → emit `drift_report.json` with `unexpected_files: []` and `missing_expected: []`
- Block the gate if `unexpected_files` non-empty without an explicit `drift-approved` comment.

---

## 6. Hard rules (mirrored from source doc)

1. **Pro-gate bypass changes never leak upstream.** Each PR must `git diff --stat` clean of: 16 `is_pro: true → false` flips, `cloud_subscribed` transform in `validation.ts`, per-card `isPro = true` overrides, `cloud_subscribed: true` injection in `loadUser`. Enforced at gate 3 (drift check).
2. **Branch hygiene.** Every upstream PR is cut off `upstream/main` directly via cherry-pick from local work — **never** off `ain`. Branch naming: `upstream/MIC-<num>-<slug>`.
3. **`bun` for JS/TS, `cargo` for Rust** (per CLAUDE.md). No `npm`, no `pnpm`.
4. **File header rule** on any new `.rs`/`.ts`/`.tsx`/`.swift`/`.py` file (per CLAUDE.md).
5. **TESTING.md regression checklist** must be read before any change touching window mgmt, tray/dock, monitors, audio, or Apple Intelligence (per CLAUDE.md). The plan/propose linear subskills already enforce this; reinforce in implementer prompts.

---

## 7. Suggested agent assignment heuristics

| Surface | Default agent | Reviewer |
|---|---|---|
| Trivial UI string / config flip | `haiku-delegate` | `code-reviewer` |
| Rust surgical fix | `claude-opus` (this session) | `code-reviewer` then Codex |
| Rust new module | `claude-opus` + `tdd-guide` | Codex (`gpt-5.3-codex`) |
| Tauri/React UI feature | `claude-opus` | `code-reviewer` |
| Cross-cutting refactor | `architect` agent first, then implementer | Codex |
| Security/OAuth change (S3 chain) | `claude-opus` + `security-reviewer` | Codex strict |
| Diagnostic CLI (S2 chain) | `claude-opus` | `code-reviewer` |
| RFC drafting (Sprint 6) | `architect` agent | Codex |

---

## 8. Open questions for Codex review

Reviewer (Codex): please challenge specifically these, in order of importance.

1. **Sprint cohesion** — Is Sprint 3's privacy track (S3-6/7/8/9) too broad to ship as one cycle? Should G-series split out as Sprint 3b?
2. **S2-7 (permission copy) placement** — Listed in Sprint 2 because it depends on S2-2. But it's also a Trust surface (closer to Sprint 3). Where does it belong?
3. **S4-7 ordering** — Per-task routing depends only on `S4-1` (types). Could it move to Sprint 3 to unblock S5-7 (clustering needs an embedding model)? Tradeoff: Sprint 4 becomes thin.
4. **`CandidateEvent` schema location** — RFC issue (S5-3) asks Rust crate vs TS-only. Recommend a position; the rest of S5 depends on it.
5. **Drift-tracking enforcement** — The "Files in scope" allowlist is hand-written. Realistic for issues like S2-2 that touch many files? Should we replace it with directory-level allowlists for larger issues?
6. **Sprint 6 graduation criteria** — RFC-only by default. What's the trigger for moving an RFC to implementation? Maintainer +1 in the upstream discussion? Two maintainer +1s? Spell it out.
7. **OOB issues (zoom, slow, onnxruntime)** — File them at the start of Sprint 1 to maximize maintainer signal time? Or batch at the end of Sprint 2 when `doctor` exists to reference?
8. **Two-model sprint review (5.2)** — Realistic at every sprint boundary, or is it overhead-heavy for Sprint 1 (which is 5 trivial PRs)? Maybe skip at Sprint 1 and start at Sprint 2.
9. **Missing items?** — `UPSTREAM_PROPOSAL.md` mentions A2b explicitly as "not a bug, do not file." Confirmed not in sprint plan. Anything else from the source doc this breakdown drops?
10. **Estimation realism** — Source doc's `Effort` columns mirrored directly. Any obvious under/over-estimates given the dependency chains we've spelled out?

---

## 9. How this will be turned into Linear (when approved)

Once Codex signs off on this doc:

1. Create the project: `linear-api.ts create-issue` with `--team MIC --title "Upstream contributions to screenpipe/screenpipe" ...` — actually a project needs the GraphQL `projectCreate` mutation (not yet wrapped in the script; add a `create-project` command if needed).
2. For each sprint, create one parent issue (Linear "Initiative" or parent issue) and child issues per row.
3. Apply labels per §1 taxonomy.
4. Set `Blocked by` / `Blocks` per §4 dependency graph.
5. Record base SHA + allowlist in each child issue body per §2 template.
6. Cycles in Linear configured sequentially; no two open simultaneously.

**Not yet implemented in `linear-api.ts`:** `create-project`, `create-cycle`, `set-blocks`, `add-label`. These will be added in a follow-up issue (suggest `MIC-tooling-1`) before the breakdown is pushed.

---

## 10. Sign-off

Reviewer (Codex), please reply with:
- `APPROVE` if the sprint shape, gates, and drift mechanics are sound.
- `CHANGES` with a numbered list answering §8 questions, plus any additional concerns.
- `REJECT` if the breakdown is fundamentally misaligned with `UPSTREAM_PROPOSAL.md`.

Trailing JSON for ingestion:

```json
{
  "doc_version": 1,
  "source_doc": "UPSTREAM_PROPOSAL.md",
  "target_team": "MIC",
  "sprint_count": 6,
  "implementation_issue_count": 38,
  "oob_issue_count": 3,
  "review_status": "pending-codex"
}
```
