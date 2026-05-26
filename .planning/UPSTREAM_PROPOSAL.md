# Upstream Contribution Plan — screenpipe/screenpipe

Audit + proposal list for PRs against `screenpipe/main`. Grouped by confidence and surface area. Citations are file paths + line numbers from the current fork (`gardinermichael/screenpipe@ain`).

This document is structured so a second-opinion reviewer (Codex / another LLM) can challenge each item independently. Each item has: **Evidence → Proposal → Risk → Effort**.

## Context & framing

- **Target repo:** `screenpipe/screenpipe` (canonical; was `mediar-ai/screenpipe` — repo moved). Roughly 18.5k stars as of May 2026; project is **active**, not abandoned. Frame PRs as polish/reliability, not rescue.
- **Themes maintainers are already shipping fixes against** (changelog + recent issues): capture/audio reliability, CPU/memory spikes, pipe session/chat duplication, install friction. Proposals that thread these themes are most likely to land. Examples from changelog: "new chat reuses an empty chat instead of spawning duplicates", credit-drain fixes on pipe presets, cleanup performance.
- **Active issue clusters (external signal — Codex web passes):**
  1. **Google integration is a live blocking bug.** Open issue `#3574 [bug] Google integration: blocked` filed 2026-05-24. Promotes B3 from "feature proposal" to "integration reliability fix with named user impact."
  2. macOS audio recording fails; Windows webcam mic not recognized; VAD model errors; OCR processing failures.
  3. Recent `onnxruntime` crash during auto-updater restart and audio/speaker model init.
  4. "App is extremely slow even on powerful hardware" — open issue with no clear repro.
  5. Intel macOS CLI Screen Recording permission problems even when native `screencapture` works. Pattern matches a broader macOS issue (e.g. Peekaboo #75) where Screen Recording reports "not granted" when spawned by Node even though permissions ARE enabled — the OS reports per-bundle-ID, not per-binary, so spawned child processes lie. Permission status must report **bundle/process identity and the actual tested call path**, not just "request permission."
  6. Privacy anxiety on Reddit: credit cards, SSNs, what gets forwarded to cloud models. Maintainer answer cites PII removal default — but the **question itself** is the signal: users can't tell what leaves their machine. Cautionary tales from Granola ("notes accidentally public") and Limitless ("pendant disappointment, support failures, lost trust") show that for an always-on memory product, privacy defaults are existential.
  7. GitHub discussion: "Is anyone using Screenpipe with different models on LM Studio?" — validates B1/B2 (first-class provider + task routing).
  8. Pipe/chat clutter — directly validates B4. Maintainers already ship in this lane.
  9. **Market gap:** Reddit thread observes Screenpipe is good passive capture, but "action after retrieval" is manual. Argues strongly for B5 (event validation), B8 (clustering), B4 (rollups), plus a new "what should I do with this?" surface (see H3, H4 below).
- **Updated priority framing (Codex):**
  1. **Diagnosability first** — browser automation handler registration, permission doctor, OAuth doctor. Turns silent failures into actionable errors.
  2. **Trust boundaries** — Google OAuth ownership (bug #3574), explicit memory destinations, local/cloud clarity.
  3. **Local AI setup** — LM Studio lifecycle-aware provider, CORS/restart copy, fallback behavior.
  4. **Timeline usability** — daily rollups, validation queue, event clustering.
  5. **Then** heavier multi-agent review.
- **Implication:** B4, C1, A4, B1 sit in active maintainer-friendly lanes. B3 is no longer "feature" — it's an open issue. The H-series (diagnosability + action layer) is new and sits between trust and timeline usability.

---

## A. Confirmed bugs (high confidence, small surgical fixes)

### A1. `get_browsers_automation_status` not registered in invoke_handler
**Status:** already patched on this fork; needs upstreaming.

- **Evidence:** `BrowserUrlCard` calls `get_browsers_automation_status()`, but the four commands were missing from `tauri::generate_handler!` in `apps/screenpipe-app-tauri/src-tauri/src/main.rs`. UI surfaced "couldn't read browser automation status". Confirmed handlers are now present at `main.rs:1066-1069`.
- **Proposal:** PR the four `permissions::*` handler registrations:
  - `check_browsers_automation_permission`
  - `request_browsers_automation_permission`
  - `get_browsers_automation_status`
  - `request_single_browser_automation`
- **Risk:** none — pure registration, no logic change.
- **Effort:** ~5 lines, one file.

### A2a. Microsoft `prompt=consent+select_account` — investigate, then fix if reproduced
**Affects 2 connectors:** Microsoft 365, Teams.

- **Evidence:** Both `auth_url` templates end with `&prompt=consent+select_account`. Inline comments in `microsoft365.rs:39` and `teams.rs:29` state intent ("second connect shows picker"). Microsoft's docs list supported `prompt` values as individual values — `login`, `none`, `consent`, `select_account`, `create` — and surface an `invalid_prompt_value` error code when something doesn't match. That strongly suggests the combined value is unsupported, but **still needs a live repro** before filing.
- **Proposal:**
  1. **Repro:** one live MS tenant, connect/disconnect/reconnect, record the HTTP exchange. If the auth response carries `error=invalid_prompt_value` or `error_description` referencing prompt, we have ground truth.
  2. If broken, change to `&prompt=consent` only. Account-picker UX, if still desired on reconnect, comes from clearing the cached account hint, not from `select_account`.
  3. **Do not touch Google connectors in the same PR** — Google supports both behaviors and the current combo may be intentional there.
- **Risk:** low once reproduced.
- **Effort:** 1 hour repro + trivial fix.

### A2b. Google connectors — **do not touch without repro**
- **Evidence:** Gmail, Google Docs, Google Sheets, Google Calendar all use `access_type=offline` + `prompt=consent select_account` (`gmail.rs:26` et al.). The combination is plausibly intentional for multi-account + refresh-token behavior on Google's tenant.
- **Action:** Leave as-is for upstream PR. If you have reproducible evidence Google misbehaves on this combo, file as separate issue with HTTP trace; until then, A2b is **not** a bug.

### A3. Tauri opener allowlist hardcodes URL schemes
**Status:** patched locally for `lmstudio://`, `cursor://`, `vscode://`, `zed://`. Two-stage upstream fix.

- **Evidence:**
  - `apps/screenpipe-app-tauri/src-tauri/capabilities/main.json` lines 62-75 hardcode every allowed scheme.
  - `connections-section.tsx:1233` opens `lmstudio://add_mcp?...` — fails silently in vanilla upstream because scheme not allowlisted. Same hazard for any future connector adding a deep-link button.
  - `notification-panel/page.tsx` uses `screenpipe://` — **also missing from the allowlist** (works because it's intercepted in-process, but native fallback would fail).
  - `tauri-plugin-deep-link` (`components/deeplink-handler.tsx:13`) handles **incoming** `screenpipe://...` callbacks. It does **not** replace outgoing-URL allowlisting in `main.json`. The two systems are orthogonal.
- **Proposal (v1 — first PR):**
  1. Add the missing schemes to `main.json`: `lmstudio://`, `cursor://`, `vscode://`, `zed://`, `screenpipe://`, `obsidian://`.
  2. Add a CI guard test using the **TypeScript Compiler API** (`typescript` is already in `apps/screenpipe-app-tauri/package.json:122` — no new dependency, no `ts-morph`):
     - Walk all `.ts`/`.tsx` files via `ts.createSourceFile`.
     - Track imports from `@tauri-apps/plugin-opener` and `@tauri-apps/plugin-shell`.
     - Find call expressions to the imported `openUrl` / `open` identifiers.
     - Extract scheme from `StringLiteral` arguments and from same-file `const FOO = "scheme://..."` literal references.
     - Compare discovered schemes against `opener:allow-open-url` in `apps/screenpipe-app-tauri/src-tauri/capabilities/main.json:62`.
     - Allow an explicit escape-hatch comment for dynamic URLs: `// opener-scheme-checked: https` on the same or preceding line.
     - Fail the build (non-zero exit) on any unannotated scheme not present in the allowlist.
  3. Document the Tauri 2.x compile-time constraint in `CONTRIBUTING.md` so the next contributor adding a deep-link button updates `main.json` in the same PR.
- **Proposal (v2 — later, if maintainers want it):**
  - Add `pub required_url_schemes: &'static [&'static str]` to `IntegrationDef`; `build.rs` codegen merges connector-declared schemes into a generated capability file. Keep v1's hand-curated entries; v2 is purely additive.
- **Risk:** v1 is near-zero. v2 only if maintainers are open to a `build.rs` step.
- **Effort:** v1 = ~1 hour. v2 = half day.

### A4. LM Studio CORS toggle has no UX warning
- **Evidence:** LM Studio's "Allow CORS" toggle only takes effect on next server restart; users assume it's live. Combined with screenpipe's Tauri webview origin being `tauri://localhost`, every new LM Studio user hits an opaque CORS error in dev tools with no in-app hint.
- **Proposal:** In the LM Studio provider settings card (see B1 below), render an inline warning: *"Toggle CORS in LM Studio's Developer tab, then **restart its local server** — changes don't apply to a running instance. screenpipe runs from `tauri://localhost`, which is a non-`http(s)` origin."*
- **Risk:** zero (UI string).
- **Effort:** 10 minutes.

---

## B. New features

### B1. LM Studio as a **lifecycle-aware AI provider**
**Correction from earlier draft:** LM Studio is already wired into the Connections UI (`connections-section.tsx:2392` registers it; `LMStudioPanel` at line 2509 detects `localhost:1234/v1/models` and offers a `lmstudio://add_mcp` deep link). What is **missing** is LM Studio as a *first-class AI preset/provider* with model **lifecycle** management — discovery, JIT load/unload, and fallback routing — not just an inference base URL.

**Critical API correction (Codex pass 3):** LM Studio's current docs use the native v1 REST API at **`/api/v1/*`** for management. It also exposes OpenAI-compatible inference at `/v1/*` and Anthropic-compatible inference at the same base. The provider needs to split **inference base URL** from **management base URL**.

**Load endpoint shape (also corrected):** the standalone load is **`POST /api/v1/models/load`** with `{ "model": "<id>", ... }` in the JSON body — **not** `POST /api/v1/models/{id}/load` as earlier drafts said. The standalone load endpoint is **non-streaming**: it returns once the model is fully loaded, with `status: "loaded"` and `load_time_seconds`. Progress is **only** surfaced via streaming `POST /api/v1/chat` with `stream: true`, which emits `model_load.start`, `model_load.progress`, `model_load.end` events when a chat call triggers an implicit load.

- **Evidence:** `AIProviderType` at `lib/utils/validation.ts:44` and the preset type in `lib/hooks/use-settings.tsx:45` lack `"lmstudio"`. Today users pick `custom` and hand-type the base URL — no model picker, no JIT load, no awareness when a model unloads.
- **Proposal:**
  1. Add `"lmstudio"` to `AIProviderType` in `validation.ts:44` and `tauri.ts:2`.
  2. New `LMStudioProviderCard` modeled on the Ollama card (`components/settings/ai-presets.tsx:1214-1230`), with **two URL fields** (or one base + auto-derive both):
     - Inference: `http://localhost:1234/v1` (OpenAI-compatible). Alternative: Anthropic-compatible endpoint if user prefers.
     - Management: `http://localhost:1234/api/v1`.
  3. Model listing from `GET /api/v1/models` — returns loaded/unloaded state per model.
  4. **Two JIT-load paths, picked deliberately:**
     - **(a) Explicit pre-load** via `POST /api/v1/models/load` with `{ "model": "<id>" }` in the body. **Non-streaming** — call returns once load is complete. UI: indeterminate progress with cancel button (load can take minutes for large models; no streaming progress is available on this endpoint per current LM Studio docs).
     - **(b) Implicit load via chat stream:** issue `POST /api/v1/chat` with `stream: true`. When the model is unloaded, the stream emits `model_load.start` → `model_load.progress` → `model_load.end` events, then continues with completion tokens. UI: surface real progress %. Use this path when the user is about to chat anyway.
  5. **Load-state awareness during inference:** if a non-streaming chat call fails with "model not loaded", auto-retry once after issuing explicit load. Surface as a toast, not a silent 500.
  6. **Optional fallback routing:** if a chosen model is unloaded *and* JIT load fails (out of VRAM, etc.), allow falling back to another configured local model with user consent.
  7. Include the CORS warning from A4.
- **Risk:** low to medium. Mirrors Ollama plus lifecycle. Connections-UI panel stays as-is (different concern: MCP wiring).
- **Effort:** ~1.5 days. Touches validation.ts, tauri.ts, use-settings.tsx (AIPreset shape), ai-presets.tsx, plus a new lifecycle service module.

> **Validates against external signal:** GitHub discussion "Is anyone using Screenpipe with different models on LM Studio?" — users want this. Combined with B2 (per-task routing), it's the local-AI story Screenpipe is currently missing.

### B2. Per-task model routing
- **Evidence:** No `embeddingModel` / `visionModel` / `chatModel` separation exists today. One model field per preset. Users running both a 70B chat model and a 1B-embedding model in LM Studio have to manually swap models in screenpipe for OCR/embedding-heavy work vs. agentic chat.
- **Proposal:** Add optional `models: { chat?: string; embedding?: string; vision?: string; summarization?: string }` on `AIPreset`. Resolver falls back to `model` field for back-compat. Surface as collapsible "advanced" panel in the preset card.
- **Risk:** medium — touches every code path that reads `preset.model`.
- **Effort:** 1-2 days. Should land *after* B1 so LM Studio benefits.

### B3. OAuth ownership mode — fixes live issue #3574 ("Google integration: blocked")
**Reframed:** this is no longer a feature proposal. Open issue `screenpipe/screenpipe#3574 [bug] Google integration: blocked` (2026-05-24) documents user impact. The right framing for upstream is **OAuth ownership mode** — a privacy/trust feature that **also** happens to be the only workable fix — not a workaround.

**Design (per Codex review):** Two pieces: (a) storage reuses existing `SecretStore`; (b) flow shape changes because current `OAuthConfig` (`crates/screenpipe-connect/src/oauth.rs:84`) explicitly assumes secrets are not in the binary and token exchange is proxied through `https://screenpi.pe/api/oauth/exchange`. User-owned credentials need a different flow.

- **Evidence:**
  - **Live bug:** issue #3574 — Google integration blocked in production builds.
  - All 4 Google connectors hardcode screenpipe's GCP `client_id`. Google has flagged that OAuth client; new users hit "This app is blocked".
  - `SecretStore` (`crates/screenpipe-secrets/src/lib.rs:5`) already encrypts in SQLite with the key in OS keychain — established pattern, used by OAuth token storage today (`oauth.rs` SecretStore integration block at lines 7-20).
  - Current proxy-exchange path assumes screenpipe's GCP client owns the secret. User-owned config can't use that proxy.
- **Proposal:**
  1. **Preferred path: PKCE installed-app flow with no client secret.** Specifically: **Desktop app** OAuth client (the official term in Google's GCP console) — *not* Web app, *not* iOS. User creates a Desktop client in their own GCP project (no secret needed), enters `client_id` in screenpipe. Token exchange and refresh both happen locally against `https://oauth2.googleapis.com/token` — no `screenpi.pe/api/oauth/exchange` round-trip needed for user-owned configs.
  2. **Fallback path (web-app client with secret):** If we must support web-app clients, store `client_secret` in `SecretStore` under key `oauth-client:<integration_id>`. Exchange happens locally, never via screenpipe's proxy.
  3. **Testing-mode expiry warning — mandatory UI copy:** Google's external OAuth consent screens in `Testing` publishing status issue refresh tokens that expire after **7 days** (except when only basic-profile scopes are requested). The setup doc and UI must warn: *"If your OAuth consent screen is set to **External + Testing**, refresh tokens may expire after 7 days. To get durable auth, either publish the consent screen, or use a Workspace/Internal setup."* Detect this surface where possible (it may show up in the refresh response error body).
  4. **Resolution order at connect time:** user override → static connector default. If user override is set, take the local-exchange path; else take the proxy path.
  5. **UI as a global mode, not a per-connector hack:** add an "OAuth ownership" section in settings with two top-level radios — `Screenpipe-managed (default)` vs `Bring your own OAuth app`. The latter reveals per-provider client config (Google / Microsoft / etc.) with `client_id` field and optional `client_secret`, plus a "How to set this up" link to a new `docs/own-oauth-credentials.md`. Frame as a **trust feature**, not a workaround.
  6. Optional follow-up: for Google Workspace org-managed accounts that can't create OAuth apps, document the Apps Script / Workspace-CLI workaround.
- **Risk:** medium. Mostly in keeping the two paths (proxy vs. local) cleanly separated in `OAuthConfig`.
- **Effort:** 2-3 days.

> **PR body framing:** lead with "Fixes #3574", then explain the design as an ownership mode. Maintainers are more likely to accept a feature that doubles as a published bug fix.

### B4. Aggregate pipe-run history by day (kills the "500 chats" flood)
- **Evidence (codex):** `apps/screenpipe-app-tauri/lib/events/pipe-run-recorder.ts:205-217` writes one `ChatConversation` per run with `kind: "pipe-run"`. Focus Assistant @ 5-min interval = 288 rows/day. No dedup, no rollup.
- **Proposal (v1 — no migration required):**
  1. New conversation kind `"pipe-run-day"` added to `ConversationKind` in `lib/hooks/use-settings.tsx`.
  2. ID shape: `pipe-run-day:<pipeName>:YYYY-MM-DD` (per pipe per day — **not** all-pipes-per-day).
  3. Recorder change: at finalize, if the pipe has `aggregate_history: "daily"`, look up the existing rollup for `(pipeName, date(now))`. If found, append messages + push `runs: PipeRunRef[]` entry; else create rollup. If not opted in, fall through to today's per-run write.
  4. Pipe manifest field `aggregate_history: "per-run" | "daily"` — **default `per-run`** for back-compat. Stock Focus Assistant ships as `daily`.
  5. Sidebar (`components/chat-sidebar.tsx:140`): render the rollup with a child-count badge ("Focus Assistant — 142 runs today"). Drill-down expands. Old `pipe-run` rows continue rendering individually with no change.
  6. **No migration in v1.** Old per-run history stays readable. Optional v2: a one-shot rollup of historical pipe-run rows behind a settings toggle.
- **Risk:** low for v1 (additive; old data untouched).
- **Effort:** 1-2 days for v1.

### B5. Event validation UI (Google-Maps-Timeline-style)
**Persistence path corrected (Codex pass 3):** earlier draft suggested adding validated fields to `SessionRecord` and persisting via `updateConversationFlags()`. That's wrong — `SessionRecord` is chat-sidebar state (`lib/stores/chat-store.ts:45`), and `updateConversationFlags()` (`lib/chat-storage.ts:400`) only writes pinned/hidden/title/browserState. Timeline events have no general candidate-event model today (see "H4/H3 dependency" note below).

- **Evidence:** No "is this correct?" affordance exists on the timeline (only speaker-identification has thumbs in `speakers-section.tsx`). The closest existing typed event is the cloud-subscription-gated `WorkflowEvent` at `crates/screenpipe-events/src/custom_events/workflow.rs:14` (fields: `event_type`, `confidence`, `description`, `activities`, `timestamp`) — not a general local store.
- **Proposal (revised):**
  1. **Introduce a new local `CandidateEvent` model** (or `ActivityCandidate` — name TBD with maintainers). Fields at minimum: `id`, `source: "workflow" | "cluster" | "meeting" | "pipe"`, `kind`, `summary`, `timeRange`, `confidence`, `validated?: "yes" | "no" | "edited"`, `validatedAt?`, `validatedNote?`, `mergedInto?: id`, `suppressedReason?`.
  2. Persist in a new `candidate-events.json` or a new SQLite table — pick whichever matches the existing storage idiom for similar event types.
  3. Inline three-button UI on each timeline candidate: ✓ correct / ✕ wrong / ✎ edit. **Do not** reuse `updateConversationFlags`.
  4. Feed validated candidates into a downstream signal later (preset training? confidence scoring?). For v1, just persist.
- **Risk:** medium — introduces a new persistence layer. Coordinate with maintainers on whether to extend `WorkflowEvent` or build local-first.
- **Effort:** 1-2 days for model + UI + persistence.

### B6. Multi-agent review of events (3-round consensus)
- **Evidence:** Event extraction today is single-pass from one LLM call.
- **Proposal:** Optional "high-confidence mode" where each candidate event is reviewed by N independent prompts (different models or different system prompts) before surfacing. Mark events as `confidence: "single-pass" | "consensus" | "disputed"`. Off by default; opt-in per pipe.
- **Risk:** cost. Make it strictly opt-in and document the token multiplier.
- **Effort:** 1-2 days.

### B7. "Hand-off to ChatGPT" export
- **Evidence:** User request — sometimes you want to review an event/run conversationally outside screenpipe.
- **Proposal:** "Open in ChatGPT" button on any conversation. Generates a self-contained prompt (events + context + user question) and either:
  - Copies to clipboard with one-click instructions, or
  - Opens `https://chatgpt.com/?prompt=...` (URL length permitting), or
  - Saves to a `.md` file the user uploads.
  Re-ingest path: paste the ChatGPT thread back into a screenpipe input that parses it into messages.
- **Risk:** low.
- **Effort:** half day.

### B8. Word-cluster event surfacing
- **Evidence:** User suggestion. Today events come from explicit LLM extraction; rare/repeated tokens across the day are not surfaced as candidate events.
- **Proposal:** Background job runs TF-IDF or KeyBERT-style clustering over the day's OCR + transcripts; surfaces top-N "topics you returned to" as candidate events with the validation UI from B5 attached. Local-only — uses an embedding model from B1/B2.
- **Risk:** depends on embedding compute budget on user hardware.
- **Effort:** 2-3 days (clustering pipeline + UI surface).

---

## C. Defaults / UX fixes

### C1. Obsidian pipe should NOT be `defaultOn: true`
- **Evidence:** `components/onboarding/pick-pipe.tsx:45` — `obsidian-daily-summary` is `defaultOn: true` in onboarding, alongside digital-clone, meeting-intel, todo-assistant. (Memories themselves don't auto-export to Obsidian — only CLAUDE.md + AGENTS.md per `screenpipe-core/src/memories/external_sync.rs:81-93`. The "mess" is purely the onboarding default.)
- **Proposal:** Flip `defaultOn: false`. Also reword the pick-pipe card so it's clear the pipe writes to a vault path the user must configure.
- **Risk:** zero.
- **Effort:** 1 line.

### C2. Make memory sync destinations explicitly opt-in
- **Evidence:** `external_sync.rs:81-93` hardcodes Claude Code + Codex destinations. New users on either tool get memories written without prompting.
- **Proposal:** First-run dialog: "screenpipe can sync memories to your dev tools — pick which: [ ] Claude Code [ ] Codex [ ] None." Persist the choice.
- **Risk:** low.
- **Effort:** half day.

### C3. Wiki-style memory explorer (replaces "shove everything into Obsidian")
- **Evidence:** User feedback that Obsidian-as-default feels presumptuous; some users have no Obsidian.
- **Proposal:** Ship an in-app memory browser — left-pane: tag/date tree; right-pane: linked-references-style explorer. Could embed [HedgeDoc](https://hedgedoc.org/) as a sidecar or build minimal native (closer to AnythingLLM's workspace browser). v1: native, minimal, read-only with full-text search. v2: edit + cross-link.
- **Risk:** scope — keep v1 tight (read + search, no edit).
- **Effort:** 3-5 days for v1.

---

## D. Needs reproduction before PRing

### D1. Zoom integration "broken" — symptom unclear
- **Evidence:** Code audit shows Zoom OAuth and refresh-token logic are well-formed (12h keep-alive loop to dodge Zoom's 15h inactivity timeout). No obvious code bug.
- **Action:** Repro the exact failure mode (token expiry? webhook? meeting fetch?) before filing. May be a config/scope issue on the screenpipe-owned Zoom OAuth app rather than code.

### D2. "Google app blocked" — server-side
- **Evidence:** Google's OAuth verification gate against screenpipe's GCP project. Local code is fine.
- **Action:** B3 (pluggable client) is the proper fix. Out-of-band: screenpipe should also pursue Google OAuth verification for its own client_id.

---

## E. Engineering hygiene (drive-by improvements)

- **E1.** `.config` file validation for CLIs — add a `screenpipe doctor` subcommand that walks every connector's expected config (~/.screenpipe/* paths, env vars, OAuth tokens, browser perms) and reports pass/fail per item. Useful for support triage. Doubles as the surface for F1-F4 diagnostics below.
- **E2.** Document Tauri's compile-time URL scheme constraint (Tauri 2.x has no runtime scheme allowlisting) prominently in CONTRIBUTING so future connectors don't repeat the `lmstudio://` failure mode.
- **E3.** Replace the four hand-maintained `is_pro` boolean fields per connector with a centralized policy (separate concerns: who can subscribe vs. what the gate does locally). Lower priority but reduces drift.

---

## F. Reliability cluster (triage, then targeted fixes)

These come from external signal (open issues, Reddit) and need **reproduction before code**. Filing them as upstream issues with a triage proposal is more valuable than guessing at fixes.

### F1. `onnxruntime` crash during auto-updater restart
- **Evidence (issue tracker):** Recent crash reports tie `onnxruntime` to auto-updater restart and audio/speaker model init.
- **Hypothesis:** Active ONNX sessions aren't being torn down before the updater calls `process::exit` / Tauri restart, so the OS reaps the process mid-inference. Or: model init races against another running session after restart.
- **Proposal:** Wrap the updater restart path with an explicit "drain in-flight ML sessions + drop ONNX `Environment`" step. Add a panic hook that logs the active model + thread state on `onnxruntime` panics so we can confirm the root cause from telemetry.
- **First step (upstream-able now):** File the diagnostic-logging PR; defer the actual fix until the panic logs identify the lifecycle order.

### F2. Audio capture reliability — macOS audio + Windows webcam mic
- **Evidence:** Multiple open issues across platforms (macOS audio recording not working, Windows webcam mic not detected).
- **Proposal:** Don't propose a fix blind. Propose a **structured device-enumeration log** + a `screenpipe doctor audio` subcommand that:
  - Lists every detected input device with vendor/product IDs.
  - Probes each for a 100ms sample.
  - Reports which device was selected and why.
- This gives maintainers a diagnostic to ship and gives users a one-liner for bug reports.

### F3. "Extremely slow on powerful hardware"
- **Evidence:** Open issue with no narrow repro. Could be FPS misconfiguration, frame backlog, OCR queue, embedding queue, or DB write contention.
- **Proposal:** Add a `screenpipe profile` mode that emits structured per-pipeline-stage timings (capture → OCR → embed → write) and a rolling p50/p95/p99 to stderr. Lets maintainers triangulate without speculation. Avoid proposing optimizations until profiling lands.

### F4. Intel macOS CLI Screen Recording permission
- **Evidence:** Reddit thread — Intel macOS users can't get `screenpipe record` to capture screens even when native `screencapture` works. Pattern matches Peekaboo #75 (macOS reports "not granted" when Node spawns a child even though permissions ARE enabled, because TCC keys off bundle ID and inheritance is partial).
- **Proposal:**
  - Add an explicit TCC entitlement check in `screenpipe record` startup that:
    1. Reports the **bundle ID** the OS sees for the current process.
    2. Reports the **parent process** chain (helps when spawned from Node / Hyper / iTerm).
    3. Calls the relevant CG/AX preflight API for the failing capability.
    4. **Performs the actual call** (e.g. one-frame screencapture) and reports the OS-level error code if it fails.
    5. Prints `tccutil reset ScreenCapture <bundle-id>` instructions on failure.
  - Document the Intel-vs-Apple-Silicon code-signing differences — dev cert / hardened runtime path can differ in ways that matter for TCC inheritance.
- This becomes the foundation for H1 (`screenpipe doctor permissions`).

### F5. Auto-updater + active recording: graceful pause
- **Adjacent to F1.** If an update fires while recording is active, screenpipe should pause capture, flush queues, restart, and resume. Spec out the lifecycle for upstream RFC before coding.

---

## G. Privacy UX — "what leaves my machine"

**Evidence:** Reddit threads asking specifically about credit cards, SSNs, and cloud-model forwarding. Maintainers correctly point to PII-removal defaults, but the question being asked means the **UX isn't conveying** what gets sent where. This is the single highest-leverage trust improvement upstream — the docs/changelog can't fix it; the UI has to.

### G1. Live data-flow inspector panel
- **Proposal:** A settings panel that shows, for each active connection/preset/pipe:
  - **What model** is processing data (provider + endpoint).
  - **What data leaves the machine** (frames? OCR text? transcripts? embeddings? metadata only?).
  - **PII redaction state** for that flow (on/off, last applied at).
  - **Where credentials live** (SecretStore / local file / proxy).
- Renders as a one-screen "audit" view. No new persistence — derived from existing settings.
- **Risk:** low. Pure read-only UI.
- **Effort:** 1-2 days.

### G2. Per-provider "what gets sent" preview before first call
- **Proposal:** First time a user enables a cloud-model preset, show a dialog: "Your screen text and audio transcripts will be sent to `api.openai.com`. PII filter is **on/off**. Continue?" One-time per provider+account.
- **Risk:** low. Modal addition.
- **Effort:** half day.

### G3. PII-filter audit log (opt-in)
- **Proposal:** When PII redaction fires, log redacted token-count + category (CC / SSN / email / etc.) to a local-only `~/.screenpipe/pii-audit.log`. User can verify the filter is actually catching things. No content logged — only metadata.
- **Risk:** low.
- **Effort:** half day.

---

## H. Diagnosability tools + action layer

New section consolidating ideas from Codex's second web pass. Splits into two themes:
- **H1-H2** — diagnostic CLIs that turn silent failures into actionable errors.
- **H3-H6** — surfaces that close the "action after retrieval" gap (the cited market complaint that Screenpipe is good at capture but acting on retrieved context is manual).

### H1. `screenpipe doctor permissions` — **macOS-first**
- **Scope decision (Codex pass 3):** ship macOS first. The concrete failure class (Peekaboo #75-style TCC false-negatives, Intel macOS Screen Recording denials, browser Automation confusion on this fork) is macOS-specific. Windows and Linux have different permission models and deserve separate platform modules.
- **macOS implementation.** For each TCC capability Screenpipe needs (Accessibility, Screen Recording, Input Monitoring, Automation, Microphone, Camera):
  - Report the **bundle ID** the OS sees for the current process.
  - Report the **parent process** chain (Node? Hyper? Finder?).
  - Report the **TCC target** (e.g. "Screenpipe Desktop controlling Brave" for Automation).
  - **Run the actual API call** that would fail in practice and report the OS-level error code, not just the preflight boolean.
  - Print exact remediation: which Settings pane to open, which `tccutil` command to run, and the exact bundle ID to scope it to.
- **Windows/Linux behavior:** the CLI shell exists from day one, but `doctor permissions` returns a **structured "not implemented yet" diagnostic** that names the platform, lists what *would* be checked when the platform module lands, and points to the issue tracker. Do **not** pretend equivalent checks exist; that's worse than no check.
- **Why this matters:** today the app shows "permission granted/not granted" booleans. The reality is that the boolean lies when parent-process context matters. Testing the call path is the only reliable signal.
- **Risk:** low. Pure read-only.
- **Effort:** 1-2 days macOS. Windows/Linux modules are follow-on PRs.

### H2. `screenpipe doctor integrations`
- **Subcommand under E1.** For each connector:
  - OAuth client identity (managed-by-Screenpipe vs user-owned).
  - Redirect URI configured.
  - Token presence + age + expiry.
  - **Live refresh test** — actually exchange the refresh token, report success/failure with the provider's error code.
  - Scopes currently granted vs. scopes the connector code requires.
  - Provider-specific warnings (e.g. "Google client is unverified; new users may see 'App blocked'").
- Pairs with B3 — once OAuth ownership mode exists, this is how users diagnose their own setup.
- **Risk:** low. Read + one network probe per connector.
- **Effort:** 1-2 days.

### H3. "Archive vs Action" UI split — **RFC**
- **Evidence:** Reddit signal that Screenpipe is read as archive/search; "what should I do with this?" is manual.
- **Proposal:** Two top-level lanes in the main UI:
  - **Archive** — current timeline, search, retrieval.
  - **Action** — candidate events surfaced by clustering (B8), validation (B5), and pipe outputs that need user decisions.
- This isn't a single PR — it's a product RFC that frames where B4, B5, B6, B8, H4 land in the overall UX.
- **Risk:** product surface.
- **Effort:** RFC first; implementation across multiple PRs.

### H4. Daily activity review inbox
- **Concrete shape of the "Action" lane from H3.** Replace pipe-run chat clutter with a "Today's candidates" queue. Each candidate event has 5 actions:
  - ✓ confirmed
  - ✕ wrong
  - ⤓ merge with another
  - ⊘ ignore (suppress similar)
  - → export (B7 ChatGPT/Claude path, or to memory destination)
- **Critical dependency (Codex pass 3):** Screenpipe does **not** have a general candidate-event store today. Existing event surfaces are fragmented:
  - `WorkflowEvent` (cloud-subscription-gated, `workflow.rs:14`)
  - Meeting/calendar events (separate)
  - UI events / pipe stream events (separate)
  - Memories (separate again)
  - `SessionRecord` (chat sidebar — not events)
  - `updateConversationFlags()` only handles pinned/hidden/title/browserState
  
  So **H4 cannot ship without the `CandidateEvent` model introduced in B5**. Decision point for maintainers: build local-first (recommended for offline-friendly UX), or extend `WorkflowEvent` (couples H4 to the cloud subscription). Recommended path: local-first model that *can ingest* `WorkflowEvent` records as one source among many.
- **Sequencing:** B5 (model + validation UI) → H4 (inbox surface) → B8 (clustering feeds candidates into the model).
- **Risk:** medium — new persistence layer, but the model is small and additive.
- **Effort:** 2-3 days for v1 (after B5's model lands).

### H5. Memory destinations as explicit connectors
- **Reframe of C2.** Today memory sync destinations are hardcoded (Claude Code, Codex). Obsidian is a default-on *pipe*. Each acts differently and there's no unified surface.
- **Proposal:** Treat **Obsidian, Claude Code, Codex, filesystem-markdown, and (future) local wiki** as first-class **memory destination connectors**, each with:
  - On/off toggle (all **off by default** for new installs).
  - Last-write timestamp + last-write preview.
  - "What gets synced" disclosure (which memory categories).
  - Per-destination format options.
- Frames memory export as user-driven, not assumed. Pairs with G1 (data-flow inspector) — destinations show up there too.
- **Risk:** medium. Touches `external_sync.rs` (`crates/screenpipe-core/src/memories/external_sync.rs:81-93`) which currently hardcodes Claude + Codex.
- **Effort:** 2-3 days.

### H6. Permission troubleshooting copy that names real bundle/process
- **Companion to H1.** Wherever the UI says "permission denied", replace generic copy with the concrete failure: *"Screenpipe Desktop (bundle: `com.screenpipe.app`) has Accessibility, but automation against Brave (bundle: `com.brave.Browser`) failed. Open System Settings → Privacy & Security → Automation → Screenpipe Desktop → enable Brave."*
- Same data H1 produces, surfaced inline at the point of failure.
- **Risk:** low.
- **Effort:** half day per failure site.

---

## Tiered roadmap (revised — diagnosability-first per Codex)

Reorganized along Codex's updated priority framing: **(1) diagnosability → (2) trust boundaries → (3) local AI setup → (4) timeline usability → (5) multi-agent review.** Each tier ships before the next.

### Tier 1 — Small fixes to push now (PRs in hours, low controversy)

| # | Item | Why now | Effort |
|---|------|---------|--------|
| 1 | **A1** — register missing browser-automation Tauri commands | Concrete bug, already locally identified, zero-risk | 5 lines |
| 2 | **C1** — flip `obsidian-daily-summary` to `defaultOn: false` | Removes unsolicited write-out surprise; one-line change | 1 line |
| 3 | **A4** — LM Studio CORS/restart warning + correct existing LM Studio panel copy | Pure UI string; addresses live user confusion | 10 min |
| 4 | **A3 v1** — add missing schemes (`lmstudio://`, `cursor://`, `vscode://`, `zed://`, `screenpipe://`, `obsidian://`) to `main.json` + CI guard test + CONTRIBUTING note | Fixes silent failure in existing deep-link buttons | 1 hr |
| 5 | **F4 (diagnostic only)** — Intel macOS CLI: log bundle ID + parent process + actual call result, with `tccutil` instructions | Pure logging addition; turns silent failure into actionable error; foundation for H1 | half day |

### Tier 2 — Diagnosability + trust boundaries (1-3 days each, ordered)

This is the highest-leverage middle tier: turns silent failures into actionable errors and addresses the bug #3574 / privacy-anxiety surfaces. Ship before any feature work.

| # | Item | Theme | Depends on | Effort |
|---|------|-------|------------|--------|
| 6 | **E1 + H1** — `screenpipe doctor` umbrella + `doctor permissions` | Diagnosability | F4 (Tier 1) | 1-2 days |
| 7 | **F1** — `onnxruntime` crash: structured panic logging on updater restart path | Diagnosability | None | 1 day |
| 8 | **F2** — `screenpipe doctor audio` (device enumeration + sample probe) | Diagnosability | E1 | 1-2 days |
| 9 | **F3** — `screenpipe profile` mode (per-stage p50/p95/p99) | Diagnosability | None | 1-2 days |
| 10 | **H6** — Permission troubleshooting copy that names real bundle/process | Diagnosability | H1 | half day per site |
| 11 | **A2a** — Microsoft OAuth `prompt=consent` fix (**repro-gated**) | Trust | Live MS tenant repro | 1 hr after repro |
| 12 | **B3** — OAuth ownership mode (closes **#3574**) | Trust | `SecretStore` (exists) | 2-3 days |
| 13 | **H2** — `screenpipe doctor integrations` (OAuth health, live refresh test) | Trust | **B3 must land first** (avoids `OAuthConfig` conflict) | 1-2 days |
| 14 | **G1** — live data-flow inspector panel | Trust | None | 1-2 days |
| 15 | **G2** — first-call "what gets sent" provider dialog | Trust | None | half day |
| 16 | **G3** — PII-filter audit log (opt-in, metadata-only) | Trust | None | half day |
| 17 | **H5** — Memory destinations as explicit connectors (Obsidian / Claude / Codex / fs-markdown, all off by default) | Trust | None | 2-3 days |

### Tier 3 — Local AI setup (1-2 days each)

| # | Item | Effort |
|---|------|--------|
| 18 | **B1** — LM Studio lifecycle-aware provider (inference @ `/v1` + management @ `/api/v1`, JIT load/unload, load-state retry, optional fallback routing) | 1.5 days |
| 19 | **B2** — per-task model routing (`chat` / `embedding` / `vision` / `summarization`) — needs RFC first if surface change is large | 1-2 days |

### Tier 4 — Timeline usability (strict ordering inside the tier)

| # | Item | Depends on | Effort |
|---|------|------------|--------|
| 20 | **B4** — daily pipe-run rollup (per-pipe-per-day, opt-in, Focus Assistant ships `daily`, no migration) | None | 1-2 days |
| 21 | **B5** — introduce `CandidateEvent` model + validation persistence (the data layer H4 writes to) | None | 1-2 days |
| 22 | **H4** — daily activity review inbox (confirm / wrong / merge / ignore / export) | **B5** must land first | 2-3 days |
| 23 | **B8** — word/link/app clustering produces `CandidateEvent` rows feeding the inbox | B5 (model) + H4 (surface) | 2-3 days |

### Tier 5 — RFCs and heavier features

These need maintainer alignment before code. File as GitHub Discussions / RFC issues first.

| # | Item | Why RFC |
|---|------|---------|
| 24 | **H3** — "Archive vs Action" top-level UI split | Frames where B4/B5/B8/H4 live; needs design alignment |
| 25 | **B6** — multi-agent verification rounds | Cost-heavy; depends on validated event data existing (Tier 4 done first) |
| 26 | **B7** — "Export prompt for external review" (ChatGPT/Claude paste-out/in) | UX pattern; small but needs agreement |
| 27 | **C3** — local wiki/memory explorer | Larger new feature; scope needs RFC |
| 28 | **F5** — auto-updater graceful pause/resume of active recording | Lifecycle design; pairs with F1 |
| 29 | **A3 v2** — `IntegrationDef::required_url_schemes` + `build.rs` codegen | Only if maintainers want it after A3 v1 lands |
| 30 | **E3** — centralize `is_pro` policy | Touches commercial gating model |

### Issues to file (not PRs)

- Zoom integration failures — file with the specific symptom (token expiry? webhook? meeting fetch?) once reproduced. Reference H2 (`doctor integrations`) as the future diagnostic path.
- "Extremely slow on powerful hardware" — file pointing at F3's `profile` mode as the diagnostic path, before anyone proposes optimizations.
- `onnxruntime` updater crash — file referencing F1's logging plan; ask if maintainers have stack traces.

### Cross-cutting "PR body framing" notes

- **Tier 1 PRs:** open against `upstream/main` directly; lead with the symptom and one-paragraph fix.
- **B3 (Tier 2 #12):** lead with "Fixes #3574". Frame as OAuth ownership mode, not workaround.
- **F-series PRs:** lead with "Adds diagnostic logging for [silent failure mode X]" and link the user-reported issue thread. Maintainers ship these readily.
- **G-series PRs:** lead with the Granola/Limitless cautionary tale ("for an always-on memory product, users need to see what leaves their machine"). Frame as trust UX.
- **H-series PRs:** for diagnostic CLIs, demonstrate the output format in the PR body so maintainers can preview the support-triage UX.

### Hard rule
**Do not include any of this fork's local Pro-gate bypass changes in upstream PRs.** The 16 `is_pro: true → false` flips, the `cloud_subscribed` transform in `validation.ts`, the per-card `isPro = true` overrides, and the `cloud_subscribed: true` injection in `loadUser` are all local-only modifications to defeat screenpipe's commercial gating. They are not bugs and not upstream-appropriate. Each PR must `git diff` clean of these files before review.

### Branch hygiene
Cut one upstream branch per Tier-1 PR off `upstream/main` directly — do **not** branch off `ain` (carries the Pro-gate bypass). Use the cherry-pick path: `git checkout upstream/main && git checkout -b upstream/<short-name> && git cherry-pick <commit>`. Open PRs against `screenpipe/screenpipe:main`.

---

## Open questions — Codex answers folded in

1. **Microsoft vs. Google prompt bug:** treat Google as unproven. A2 split into A2a (Microsoft, repro-gated) and A2b (Google, leave alone).
2. **B4 rollup shape:** per-pipe-per-day with ID `pipe-run-day:<pipeName>:YYYY-MM-DD`. Opt-in per pipe. Focus Assistant stock-configures as `daily`.
3. **B3 storage:** reuse `SecretStore` (`crates/screenpipe-secrets/src/lib.rs`). Connector-scoped key like `oauth-client:<integration_id>`. Resolution: user override first, static default second.
4. **Tauri deep-link vs. opener:** orthogonal — deep-link is incoming, opener is outgoing. A3 stays opener-focused; no pivot.
5. **B5/B6/B8 PR count:** three. Validation UI lands first; clustering second (depends on the surface from B5); multi-agent review last (cost-heavy, depends on validated-event signal existing).

---

## Resolved by Codex pass 3 (for the record)

All six pass-2 unknowns now have concrete answers folded into the relevant sections:

1. ✅ **A3 guard test mechanics** — TypeScript Compiler API (already in deps), no `ts-morph`, no regex. Specific walk + escape-hatch comment documented in A3 v1.
2. ✅ **B1 LM Studio load** — endpoint is `POST /api/v1/models/load` with body, **not** `/api/v1/models/{id}/load`. Standalone load is non-streaming. Progress only via streaming `/api/v1/chat`. Both paths now documented in B1.
3. ✅ **B3 token refresh** — local exchange via `oauth2.googleapis.com/token` works for Desktop client + PKCE. **Mandatory testing-mode 7-day-expiry warning** added to B3.
4. ✅ **H1 OS scope** — macOS-first. Windows/Linux return structured "not implemented yet" diagnostic on day one.
5. ✅ **H4/H3 events dependency** — Screenpipe has no general candidate-event store. `WorkflowEvent` is cloud-only; `SessionRecord` is chat state; `updateConversationFlags` is wrong target. **B5 now introduces a new `CandidateEvent` model.** H4 depends on it.
6. ✅ **Microsoft prompt** — MS docs imply combined value is unsupported (`invalid_prompt_value` error code). Still repro-gated, but suspicion strong enough to file A2a immediately after one tenant test.

## New unknowns (worth a fourth pass)

These emerged from pass 3's corrections:

- **`CandidateEvent` schema specifics:** which fields are strictly required vs. optional? Should `source` be an enum or a free-form string for extensibility? Should the model live in a Rust crate (`screenpipe-events`?) and surface via Tauri commands, or in TypeScript only for now? Recommended: file as an RFC issue and let maintainers pick.
- **Where to surface H1's "tested call path" results:** CLI output is straightforward, but does the UI also surface them in the existing permission/diagnostic panel? That panel currently shows booleans — replacing them with the richer doctor output is its own UX decision.
- **B1 implicit-load chat-streaming consistency:** if the user is *not* about to chat (e.g. selecting a model from the preset settings page), only path (a) explicit-load is available, and it's indeterminate. Acceptable UX, or do we need a workaround (e.g. issue a 1-token "warmup" chat stream to get the progress events)? Worth a small probe before committing.
- **OAuth ownership mode interaction with existing tokens:** when a user switches from `Screenpipe-managed` to `Bring your own`, what happens to existing refresh tokens? Reset all? Migrate? Coexist with provider-tagged labels? Spell out the migration before B3 implementation.
- **PR ordering within Tier 2:** F-series (logging) is purely additive, but B3 (OAuth ownership) touches `OAuthConfig` and may conflict with H2 (`doctor integrations`). Recommend B3 → H2 strictly, not in parallel.
