# Pro-Gate Bypass Exclusion Checklist

## Why this exists

The `ain` branch carries fork-only commercial gateway bypass changes that **must never leak into upstream PRs**. Upstream maintainers will reject any PR carrying these markers. Gate 3 (drift check) in the issue template is the automated enforcement; this document tells reviewers what patterns to look for and how to verify clean diffs before opening a PR.

## What to look for

The four primary bypass patterns found on `ain`:

1. **`is_pro: false` → `true` flips** across 30+ connector definitions
   - File glob: `crates/screenpipe-connect/src/connections/*.rs`
   - Search pattern: `is_pro: [a-z]*,?\s*};` where the value changes from local default to `true`

2. **`cloud_subscribed` transform in validation schema**
   - File: `apps/screenpipe-app-tauri/lib/utils/validation.ts:40`
   - Pattern: `.transform(() => true)` on the `cloud_subscribed` field, forcing all parsed user objects to `cloud_subscribed === true` unconditionally

3. **Per-card `isPro = true` overrides in component files**
   - File glob: `apps/screenpipe-app-tauri/components/settings/*-card.tsx`
   - Examples found: `gmail-card.tsx:24`, `google-calendar-card.tsx:47`, `google-docs-card.tsx:32`, `google-sheets-card.tsx:28`
   - Pattern: `const isPro = true; // LOCAL OVERRIDE: bypass per-card Pro gate`

4. **`cloud_subscribed: true` injection in `loadUser` and settings state**
   - File glob: `apps/screenpipe-app-tauri/lib/hooks/use-settings.tsx`, `src-tauri/src/store.rs`
   - Patterns: `user: { ...settings.user, cloud_subscribed: true }` or `cloud_subscribed: Some(true)` where values are hardcoded

## How to verify before opening a PR

Run these commands from the worktree (with base SHA `dcfda38c0a0191c133d322a695ad05c96f483aa1` as the anchor):

1. **Diff against allowed scope:**
   ```bash
   git diff dcfda38c0a0191c133d322a695ad05c96f483aa1 --name-only | \
     while read f; do
       if [[ "$f" =~ ^(crates/screenpipe-connect|apps/screenpipe-app-tauri/components|apps/screenpipe-app-tauri/lib) ]]; then
         echo "CHECK: $f";
       fi;
     done
   ```

2. **Grep for `is_pro` mutations:**
   ```bash
   git diff dcfda38c0a0191c133d322a695ad05c96f483aa1 -- 'crates/screenpipe-connect/src/connections/*.rs' | \
     grep -E '^\+.*is_pro:\s*(true|false)' | grep -v '^+' && echo "FAIL: is_pro mutation found" || echo "PASS: no is_pro mutations"
   ```

3. **Grep for `cloud_subscribed` transform injection:**
   ```bash
   git diff dcfda38c0a0191c133d322a695ad05c96f483aa1 -- 'apps/screenpipe-app-tauri/lib/utils/validation.ts' | \
     grep -E '\.transform\(\(\)\s*=>\s*true\)' && echo "FAIL: cloud_subscribed transform found" || echo "PASS: no transform injection"
   ```

4. **Grep for per-card `isPro = true` overrides:**
   ```bash
   git diff dcfda38c0a0191c133d322a695ad05c96f483aa1 -- 'apps/screenpipe-app-tauri/components/settings/*-card.tsx' | \
     grep -E '^\+.*isPro\s*=\s*true' && echo "FAIL: per-card override found" || echo "PASS: no per-card overrides"
   ```

5. **Grep for `cloud_subscribed` hardcoding:**
   ```bash
   git diff dcfda38c0a0191c133d322a695ad05c96f483aa1 -- \
     'apps/screenpipe-app-tauri/lib/hooks/use-settings.tsx' \
     'apps/screenpipe-app-tauri/src-tauri/src/store.rs' | \
     grep -E '(cloud_subscribed:\s*true|cloud_subscribed:\s*Some\(true\))' && \
     echo "FAIL: cloud_subscribed hardcoding found" || echo "PASS: no hardcoded subscriptions"
   ```

All five must print "PASS" before the PR is review-ready.

## What to do if a hit appears

**Abort the worktree.** Do not attempt inline cleanup. The correct fix is:

1. Stop the current work branch.
2. Start a fresh checkout from `upstream/main`:
   ```bash
   git fetch upstream
   git checkout -b upstream/<feature-name> upstream/main
   ```
3. Cherry-pick **only the clean commits** from the abandoned worktree, or re-implement from scratch if the commit history is entangled.
4. Re-run the five checks above.
5. Only then proceed to review and push.

Attempting to "surgically remove" bypass changes mid-stream is error-prone and is more time-consuming than starting fresh.
