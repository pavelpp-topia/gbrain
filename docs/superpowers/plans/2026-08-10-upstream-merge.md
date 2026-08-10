# Upstream Merge (topia-main ← origin/master) Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to run this task-by-task. This is an integration/merge task, not new feature development, so tasks are conflict-resolution + verification steps rather than TDD red/green cycles. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring `topia-main` up to date with `origin/master` (garrytan/gbrain upstream, currently 532 commits ahead of our merge-base) while preserving all 3 of our fork-only feature sets, and dropping/adopting upstream equivalents where they already cover something we built.

**Architecture:** One `git merge origin/master` performed in an isolated worktree branch (not directly on `topia-main`), resolved file-by-file using the specific diffs identified below, validated with typecheck + full test suite + targeted tests for our 3 features, then fast-forwarded into `topia-main` and pushed to `topia` (bitbucket) only after explicit confirmation.

**Tech Stack:** Bun, TypeScript, git.

---

## Background / research already done

`topia-main` diverged from upstream at merge-base `5008b287` (upstream v0.42.59.0). Since then:
- `origin/master` (now v0.43.0.0) gained **532 commits / 896 files / +84,960 −5,744 lines**.
- `topia-main` gained **17 commits / 16 files / +1,324 −25 lines**, entirely on top of merge-base.

`topia-main`'s 17 commits reduce to exactly **3 fork-only feature sets**:

1. **Self-hosted TEI / Jina embedding recipe + token-split batching** (the bulk of the work: `src/core/ai/recipes/jina.ts` [new file, no upstream equivalent], `src/core/ai/gateway.ts`, `build-gateway-config.ts`, `types.ts`, `recipes/index.ts`, `src/core/config.ts`, plus tests and 2 plan/spec docs under `docs/plans/` and `docs/specs/`, plus `docs/mcp/DEPLOY.md`). **Not implemented upstream** — `git grep jina` on `origin/master` returns nothing. Must be preserved in full.
2. **`'*'` wildcard in `federated_read`** (`src/core/operations.ts` — new `ALL_SOURCES_WILDCARD` export + checks in `sourceScopeOpts`/`resolveRequestedScope`; `src/core/legacy-token-scope.ts`; `src/commands/auth.ts`; tests `test/get-page-federated-scope.test.ts`, `test/legacy-token-federated-scope.test.ts`, `test/source-scope-resolver.test.ts`). **Not implemented upstream** — upstream did heavily rework the surrounding `allowedSources`/federated-read machinery in `operations.ts` (+1237/−205 lines) but has no `'*'`-wildcard concept. Must be preserved, re-integrated into upstream's reworked functions.
3. **`last_commit` exposed in `sources list`** — we added this (`ede6ac0f`) then reverted it ourselves (`882b64d4`), net **zero diff** on our side. Upstream's `src/commands/sources.ts` already exposes `last_commit` today. **No action needed** — just take upstream's version, nothing to reconcile.

Every file `topia-main` touches that also changed upstream (the real conflict surface, confirmed by diffing both sides against the merge-base):

| File | Our diff | Upstream diff | Risk |
|---|---|---|---|
| `src/core/ai/gateway.ts` | +107/−17 | +783/−90 | **High** — upstream substantially reworked this file |
| `src/core/config.ts` | +18/−1 | +208/−1 | Medium |
| `src/core/operations.ts` | +24/−2 | +1237/−205 | **High** — same functions we touched were reworked upstream |
| `src/commands/auth.ts` | +9/−0 (approx) | +149/−4 | Medium |
| `src/core/legacy-token-scope.ts` | +41 (net incl. wildcard) | +41 (unrelated additions) | Low — non-overlapping additions, easy union |
| `src/core/ai/build-gateway-config.ts` | small | +52/−7 | Low-Medium |
| `src/core/ai/types.ts` | small | +59/−3 | Low |
| `src/core/ai/recipes/index.ts` | +2 lines (jina import/register) | +28/−2 lines (4 new recipes) | **Low** — confirmed trivial, both sides append independent lines |
| `test/ai/adaptive-embed-batch.test.ts` | +202 (new) | +91/−? | Medium |
| `test/get-page-federated-scope.test.ts` | +8 | +152 | Medium |
| `test/legacy-token-federated-scope.test.ts` | +9 (new) | +61 (new) | Medium |
| `docs/mcp/DEPLOY.md` | +28 (new) | +59 | Low |
| `test/source-scope-resolver.test.ts` | +52 (new) | untouched upstream | None — clean add |
| `docs/plans/2026-06-17-*.md`, `docs/specs/2026-06-17-*.md` | new | untouched upstream | None — clean add |

`VERSION` / `package.json` / `CHANGELOG.md`: **not touched by any `topia-main`-only commit**, so the 3-way merge auto-resolves to upstream's values (`0.43.0.0`) with no conflict. This is a private fork sync, not a `/ship` release of our own — do NOT run the CLAUDE.md release-versioning ceremony for this merge; just accept upstream's version files as-is.

`CLAUDE.md` itself: not touched by `topia-main` — take upstream's wholesale, no manual merge.

**Strategy chosen: merge, not rebase.** Rebasing 17 commits (2 of which are a pure add+revert no-op) across 532 upstream commits would replay conflict resolution multiple times for zero benefit over a single 3-way merge. Merge it once.

---

### Task 0: Snapshot + isolated workspace

**Files:** none (git only)

- [ ] **Step 1: Confirm clean tree and note current head**

```bash
git status
git rev-parse topia-main > /tmp/topia-main-pre-merge.txt
cat /tmp/topia-main-pre-merge.txt
```
Expected: `nothing to commit, working tree clean`; a 40-char SHA saved (this is our rollback point).

- [ ] **Step 2: Refresh remotes**

```bash
git fetch origin
git fetch topia
```

- [ ] **Step 3: Create an isolated worktree for the merge**

```bash
git worktree add ../gbrain-upstream-merge -b topia-main-upstream-merge topia-main
cd ../gbrain-upstream-merge
```
Do all work below inside this worktree. `topia-main` itself is untouched until Task 6.

---

### Task 1: Perform the merge and triage conflicts

**Files:** whatever git reports as conflicted

- [ ] **Step 1: Run the merge**

```bash
git merge origin/master --no-commit --no-ff
```
Expected: exits non-zero with conflicts (do not use `--no-edit`/auto-commit — we need to inspect before committing).

- [ ] **Step 2: List conflicted files and confirm they match the predicted surface**

```bash
git status --porcelain | grep '^UU\|^AA\|^DD'
```
Expected: a subset of the file list in the table above. If a file conflicts that is NOT in that table, stop and investigate — it means our diffstat analysis missed something (re-run `git diff <merge-base> topia-main -- <file>` against that specific file before resolving).

---

### Task 2: Resolve `src/core/ai/recipes/index.ts` (low risk)

**Files:** `src/core/ai/recipes/index.ts`

- [ ] **Step 1: Open the conflict and keep BOTH sides' additions**

Ours added:
```ts
import { jina } from './jina.ts';
```
in the import block, and `jina,` in the `ALL` array.

Upstream added 4 new recipes (`claudeCli`, `dashscopeRerank`, `moonshot`, `mistral`, `nvidia`, `perplexity`) plus a test-only seam (`_testRecipes` / `__setTestRecipesForTests`) with `getRecipe`/`listRecipes` changed to consult it.

Resolve by keeping upstream's full file verbatim, then re-adding our two lines:
- `import { jina } from './jina.ts';` alongside the other recipe imports
- `jina,` inside the `ALL` array (after `zeroentropyai,` is fine, order doesn't matter)

- [ ] **Step 2: Verify no duplicate identifiers**

```bash
grep -c "jina" src/core/ai/recipes/index.ts
```
Expected: `2` (one import, one array entry).

- [ ] **Step 3: Stage**

```bash
git add src/core/ai/recipes/index.ts
```

---

### Task 3: Resolve `src/core/ai/recipes/jina.ts` (should be a clean add, verify)

**Files:** `src/core/ai/recipes/jina.ts`

- [ ] **Step 1: Confirm this file is not touched upstream and applies cleanly**

```bash
git status --porcelain -- src/core/ai/recipes/jina.ts
```
Expected: no output (git auto-adds it since only our side created it) OR `A  src/core/ai/recipes/jina.ts` with no conflict markers. If it shows `UU`, upstream independently created a file at the same path — open it and manually pick our implementation, renaming upstream's if it's an unrelated recipe.

---

### Task 4: Resolve `src/core/ai/gateway.ts`, `build-gateway-config.ts`, `types.ts`, `src/core/config.ts` (the token-split / Jina feature)

**Files:** `src/core/ai/gateway.ts`, `src/core/ai/build-gateway-config.ts`, `src/core/ai/types.ts`, `src/core/config.ts`

- [ ] **Step 1: Re-derive exactly what our side changed in each file, in isolation**

```bash
git diff 5008b287e47bf791132eedfebf66bdef11e9398c topia-main -- src/core/ai/gateway.ts > /tmp/ours-gateway.diff
git diff 5008b287e47bf791132eedfebf66bdef11e9398c topia-main -- src/core/ai/build-gateway-config.ts > /tmp/ours-build-gateway-config.diff
git diff 5008b287e47bf791132eedfebf66bdef11e9398c topia-main -- src/core/ai/types.ts > /tmp/ours-types.diff
git diff 5008b287e47bf791132eedfebf66bdef11e9398c topia-main -- src/core/config.ts > /tmp/ours-config.diff
```
Read each file. These are the exact hunks that must survive, semantically, in the merged file — the goal is NOT to keep the literal diff hunk (upstream reworked surrounding code) but to reproduce the same *behavior*: per-text truncation to `maxBatchTokensOverride`, `GBRAIN_EMBED_MAX_CHARS_PER_TEXT` char cap, concurrent sub-batch dispatch, `embed.http_concurrency` config key (and removal of the old `_embedTuning` env vars), and the semaphore-scope comment.

- [ ] **Step 2: For each conflicted file, start from upstream's version and reapply our behavior on top**

For each file: take upstream's full post-merge content as the base (`git show origin/master:<path> > <path>` if the conflict markers are too tangled to edit in place), then manually reapply each hunk from the corresponding `/tmp/ours-*.diff`, adapting line numbers/context to upstream's refactored surroundings. Do NOT try to keep upstream's version untouched and bolt ours on side-by-side if the functions were renamed/restructured — read upstream's new function shape first, then insert our logic at the equivalent point.

- [ ] **Step 3: Grep for our env-var / config-key surface to confirm nothing got dropped**

```bash
grep -rn "GBRAIN_EMBED_MAX_CHARS_PER_TEXT\|embed.http_concurrency\|maxBatchTokensOverride" src/core/ai/ src/core/config.ts
```
Expected: matches in `gateway.ts` and `config.ts` (and possibly `build-gateway-config.ts`/`types.ts` for the type/threading of `maxBatchTokensOverride`).

- [ ] **Step 4: Stage**

```bash
git add src/core/ai/gateway.ts src/core/ai/build-gateway-config.ts src/core/ai/types.ts src/core/config.ts
```

---

### Task 5: Resolve `src/core/operations.ts`, `src/core/legacy-token-scope.ts`, `src/commands/auth.ts` (the `'*'` wildcard federated-read feature)

**Files:** `src/core/operations.ts`, `src/core/legacy-token-scope.ts`, `src/commands/auth.ts`

- [ ] **Step 1: Re-derive our exact wildcard diff in isolation (already captured above, re-save for convenience)**

```bash
git diff 5008b287e47bf791132eedfebf66bdef11e9398c topia-main -- src/core/operations.ts src/core/legacy-token-scope.ts src/commands/auth.ts > /tmp/ours-wildcard.diff
```

- [ ] **Step 2: `legacy-token-scope.ts` — trivial union, no semantic overlap**

Upstream added two brand-new, unrelated functions (`parseTakesHoldersAllowList`, `coerceLegacyPermissions`) appended to the file. We modified the existing `parseLegacyTokenScope` function body + added an import of `ALL_SOURCES_WILDCARD` from `./operations.ts`. Resolve by: keep upstream's new functions untouched, replace `parseLegacyTokenScope`'s body with our version (the one that computes `floor = allowedSources.find(s => s !== ALL_SOURCES_WILDCARD) ?? 'default'`), and add the `import { ALL_SOURCES_WILDCARD } from './operations.ts';` line at the top.

- [ ] **Step 3: `operations.ts` — re-integrate the wildcard into upstream's reworked functions**

This is the highest-attention file. Upstream reworked the `allowedSources`/federated-read machinery substantially (same functions we touched: `sourceScopeOpts`, `resolveRequestedScope`, plus the `OperationContext` interface). Do this in order:
1. Add the `ALL_SOURCES_WILDCARD` export (our new constant + its doc comment) near upstream's `OperationContext` interface — pick a spot after the interface, matching where we originally placed it.
2. In upstream's (possibly renamed/restructured) `sourceScopeOpts`, add the wildcard short-circuit as the FIRST check inside the `allowedSources` branch: `if (allowed && allowed.includes(ALL_SOURCES_WILDCARD)) return {};` — before any other array-based scoping logic, per our comment ("Checked BEFORE the generic array branch so it returns the no-filter shape").
3. In upstream's (possibly restructured) `resolveRequestedScope`, add the `wildcard` check and use it to skip the out-of-grant `permission_denied` throw, exactly as our diff shows.
4. Read upstream's new federated-read code fully first (`sourceScopeOpts`, `resolveRequestedScope`, and anywhere else `allowedSources` is consumed for a read-scope decision, e.g. `linkReadScopeOpts` per our diff) — if upstream introduced NEW call sites that also do `allowedSources`-based scoping (beyond the two functions we originally touched), add the same wildcard short-circuit there too, since a partial wildcard would be a silent read-scope hole.

- [ ] **Step 4: `auth.ts` — trivial, cosmetic-only**

Just the CLI usage-string and the `Federated reads:` printed line. Reapply verbatim from `/tmp/ours-wildcard.diff` onto upstream's version of `registerClient` (check the surrounding function didn't get renamed/restructured — upstream added +149 lines to this file, likely new subcommands, not necessarily touching `registerClient` itself; confirm with `grep -n "function registerClient" src/commands/auth.ts` before assuming line numbers).

- [ ] **Step 5: Stage**

```bash
git add src/core/operations.ts src/core/legacy-token-scope.ts src/commands/auth.ts
```

---

### Task 6: Resolve remaining test files + docs

**Files:** `test/ai/adaptive-embed-batch.test.ts`, `test/get-page-federated-scope.test.ts`, `test/legacy-token-federated-scope.test.ts`, `docs/mcp/DEPLOY.md`

- [ ] **Step 1: For each, diff our side and upstream's side against merge-base separately, then union manually (same pattern as Task 4/5)**

```bash
git diff 5008b287e47bf791132eedfebf66bdef11e9398c topia-main -- test/ai/adaptive-embed-batch.test.ts test/get-page-federated-scope.test.ts test/legacy-token-federated-scope.test.ts docs/mcp/DEPLOY.md > /tmp/ours-tests-docs.diff
git diff 5008b287e47bf791132eedfebf66bdef11e9398c origin/master -- test/ai/adaptive-embed-batch.test.ts test/get-page-federated-scope.test.ts test/legacy-token-federated-scope.test.ts docs/mcp/DEPLOY.md > /tmp/theirs-tests-docs.diff
```
These are test files, so err on the side of keeping BOTH sides' test cases (more coverage, not less) rather than picking one. For `DEPLOY.md`, keep both sections; if they describe the same deployment step differently, prefer upstream's wording but keep any Jina/self-hosted-TEI-specific instructions we added.

- [ ] **Step 2: Stage**

```bash
git add test/ai/adaptive-embed-batch.test.ts test/get-page-federated-scope.test.ts test/legacy-token-federated-scope.test.ts docs/mcp/DEPLOY.md
```

---

### Task 7: Confirm no unresolved conflicts remain, commit the merge

**Files:** none

- [ ] **Step 1: Verify clean**

```bash
git status --porcelain | grep '^UU\|^AA\|^DD'
```
Expected: no output.

- [ ] **Step 2: Sanity-check VERSION/package.json/CHANGELOG per the CLAUDE.md merge-conflict-recovery habit (should already be conflict-free, but verify no stray markers)**

```bash
grep -rn "^<<<<<<<\|^=======$\|^>>>>>>>" VERSION package.json CHANGELOG.md 2>/dev/null
echo "VERSION:      $(cat VERSION)"
echo "package.json: $(node -e 'process.stdout.write(require("./package.json").version)')"
grep -E "^## \[" CHANGELOG.md | head -1
```
Expected: no conflict markers found; all three show `0.43.0.0` (or whatever origin/master's current tip is by the time you run this).

- [ ] **Step 3: Full repo-wide conflict-marker sweep (belt and suspenders)**

```bash
grep -rln "^<<<<<<<\|^=======$\|^>>>>>>>" --include='*.ts' --include='*.md' --include='*.json' src/ test/ docs/ 2>/dev/null
```
Expected: no output.

- [ ] **Step 4: Commit the merge**

```bash
git commit
```
Use the default merge commit message git prepares (lists both parent tips); do not `--no-edit` blindly — glance at it first.

---

### Task 8: Install deps + typecheck

**Files:** none

- [ ] **Step 1: Reinstall (upstream's 532 commits almost certainly touched `package.json` deps)**

```bash
bun install
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck > /tmp/merge_typecheck.txt 2>&1
echo "EXIT=$?"
tail -80 /tmp/merge_typecheck.txt
```
Expected: `EXIT=0`. If not, the errors will point directly at any spot where our reapplied logic doesn't match upstream's new types (most likely in `gateway.ts`/`types.ts`/`operations.ts`). Fix and re-run until clean — per the CLAUDE.md engine-parity + contract-first invariants, do not silence a type error with `any`; find the real shape mismatch.

---

### Task 9: Full test suite

**Files:** none

- [ ] **Step 1: Run the full suite, capturing to a file first (per CLAUDE.md's iron rule — never pipe test output through `tail` directly)**

```bash
bun test > /tmp/merge_tests.txt 2>&1
echo "EXIT=$?"
tail -80 /tmp/merge_tests.txt
grep -E '(fail\)|✗|error:)' /tmp/merge_tests.txt | head -50
```

- [ ] **Step 2: Triage any failures**

For each failure, classify: (a) pre-existing upstream flake/failure unrelated to our merge — verify by checking if `origin/master` alone (before merge) also fails that test; (b) a real conflict-resolution mistake in Tasks 2–6 — go back and fix the specific file; (c) a genuine behavior change upstream that our feature needs to adapt to — fix forward, don't revert upstream's change.

- [ ] **Step 3: Run our 3 features' targeted tests explicitly, even if the full suite passed, since they're the highest-value regression surface**

```bash
bun test test/ai/adaptive-embed-batch.test.ts test/get-page-federated-scope.test.ts test/legacy-token-federated-scope.test.ts test/source-scope-resolver.test.ts > /tmp/merge_feature_tests.txt 2>&1
echo "EXIT=$?"
cat /tmp/merge_feature_tests.txt
```
Expected: `EXIT=0`, all our wildcard + Jina/token-split tests pass against upstream's reworked code.

- [ ] **Step 4: Run any repo-provided combined gate if present**

```bash
grep -n '"ci:local"' package.json
```
If present, run `bun run ci:local > /tmp/merge_ci_local.txt 2>&1; echo "EXIT=$?"; tail -100 /tmp/merge_ci_local.txt` and confirm `EXIT=0`.

---

### Task 10: Manual smoke check of the 2 preserved features

**Files:** none (manual verification, not automated — do this because the tests were written against the OLD gateway.ts internals and might pass while missing a real runtime break)

- [ ] **Step 1: Confirm the Jina recipe is still registered and reachable**

```bash
bun run -e "import { getRecipe, listRecipes } from './src/core/ai/recipes/index.ts'; console.log(!!getRecipe('jina'), listRecipes().map(r => r.id));"
```
Expected: `true` printed, and `jina` present in the listed recipe ids alongside upstream's new ones (`claude-cli`, `dashscope-rerank`, `moonshot`, `mistral`, `nvidia`, `perplexity`, etc.).

- [ ] **Step 2: Confirm the wildcard constant + short-circuit are wired**

```bash
grep -n "ALL_SOURCES_WILDCARD" src/core/operations.ts src/core/legacy-token-scope.ts
```
Expected: the export, the `sourceScopeOpts` check, and the `resolveRequestedScope` check are all present (3+ matches), plus the import in `legacy-token-scope.ts`.

---

### Task 11: Fast-forward `topia-main`, clean up, and push (only with explicit go-ahead)

**Files:** none

- [ ] **Step 1: Return to the primary worktree and fast-forward `topia-main` to the validated merge**

```bash
cd /Users/pavel/IdeaProjects/gbrain
git merge --ff-only topia-main-upstream-merge
```
Expected: fast-forwards cleanly since `topia-main-upstream-merge` was branched directly from `topia-main`'s tip.

- [ ] **Step 2: Remove the worktree and temp branch**

```bash
git worktree remove ../gbrain-upstream-merge
git branch -d topia-main-upstream-merge
```

- [ ] **Step 3: STOP — do not push yet.** Report back: typecheck/test results, the merge commit SHA, and a summary of anything non-mechanical you had to decide during conflict resolution (e.g. if upstream added new `allowedSources` call sites in Task 5 Step 4 that needed a wildcard check). Push to `topia` (bitbucket, the shared team remote) only after the user explicitly confirms — this rewrites the shared branch's history graph (adds a merge commit) and should not happen silently.

```bash
# Only after explicit user go-ahead:
git push topia topia-main
```

---

### Task 12: Post-merge adoption opportunities (run after Task 11, on `topia-main`, as follow-up commits — not part of the merge commit itself)

These are upstream features from the 532-commit range worth deliberately turning on for our deployment (Postgres/RDS engine, self-hosted TEI/Jina embeddings, multi-source Confluence+Bitbucket code KB on EKS). None of them block landing the merge — do the merge, validate it (Tasks 0–11), commit and push it, THEN come back to this task as separate, individually-reviewable follow-up commits.

**Files:** none yet — this task is investigation + a go/no-go per item, not a spec for the implementation

- [ ] **Step 1: Investigate `--surface verbs` (MEMORY_VERBS v1) for our OpenClaw/agent connections**

Read `docs/protocol/MEMORY_VERBS_v1.md` (added upstream) in full first. This introduces a frozen 5-verb MCP surface (`recall`/`remember`/`entity`/`synthesize`/`forget`) as an alternative to exposing our full ~90-op catalog to a connected coding agent — opt-in via `gbrain serve --surface verbs` (existing `--surface full` installs are unchanged, no schema migration).

```bash
gbrain protocol --json | head -100          # inspect the frozen spec
gbrain serve --surface verbs &              # try it against a scratch/dev brain first, NOT prod
gbrain protocol conformance                 # self-certify our server against the frozen contract
```

Evaluate against how our OpenClaw deployment currently connects (check the Confluence deployment doc — [[gbrain-mcp-confluence-docs]] memory — for the current `claude mcp add` / connection config). Decide: keep `--surface full` (agents keep the full op catalog we may already depend on for code-search-specific ops like `code_def`/`code_blast`/`code_refs`) vs. adding a SEPARATE `--surface verbs` endpoint alongside it for a simpler onboarding path for new agent integrations. **Do not swap the existing production connection to `--surface verbs` without first confirming none of our current agent workflows depend on ops outside the 5-verb set** (code-graph ops especially — the verbs surface is oriented at general memory recall, not code-indexing operations).

Write findings as a short note; if adopting, that's its own follow-up plan/PR, not squeezed into this merge.

- [ ] **Step 2: Investigate `--bound-slug-prefixes` as the write-side complement to our `'*'` wildcard**

We built read-scope wildcard federated-read (`ALL_SOURCES_WILDCARD`, Task 5 above). Upstream's `--bound-slug-prefixes` (v0.42.72.0) is the write-side isolation primitive — confines a registered client to a slug prefix and refuses (rather than silently unfences) any op that can't be prefix-checked. If we have or plan multi-tenant / per-team write access to the code KB, this is directly applicable.

```bash
grep -n "bound-slug-prefixes\|bound_slug_prefixes\|boundSlugPrefixes" src/commands/auth.ts src/core/operations.ts
gbrain auth register-client --help | grep -A2 bound
```
Decide whether any of our currently-registered OAuth clients would benefit from a bound prefix; if so, file it as its own follow-up (a `rescope-client` command, no re-registration/secret-rotation needed per the CHANGELOG).

- [ ] **Step 3: Run `apply-migrations --yes` once, deliberately, post-merge**

v0.42.70.0 fixed a bug where `apply-migrations --yes` printed "All migrations up to date" with exit 0 while actually skipping real migrations. If our brain has been silently carrying a stale schema because of this bug on an older binary, this is the one command that surfaces and fixes it.

```bash
gbrain doctor
gbrain apply-migrations --yes
gbrain doctor    # re-run, confirm no "schema behind" finding remains
```

- [ ] **Step 4: Diff our custom embed/token-split code against upstream's overlapping self-hosted-embedding fixes**

v0.42.65.0 added an `llama-server` 32-input batch cap and Ollama Matryoshka-dimension threading; v0.42.69.0 fixed self-hosted OpenAI-compatible servers rejecting an explicit `dimensions` param when width already matches native width. None of these are Jina-recipe-specific, but they touch the same `src/core/ai/gateway.ts` machinery our token-split code lives in (Task 4). After the merge lands and tests pass, specifically re-read the merged `gateway.ts` for whether our per-text truncation / concurrent dispatch logic should apply the same "don't send `dimensions` if it already matches native width" guard for the Jina recipe, and whether our sub-batch sizing should respect an analogous hard cap the way `llama-server`'s does.

- [ ] **Step 5: Confirm Sonnet 5 / Fable 5 pricing landed and is being used**

v0.42.61.0 added these to the canonical pricing table (`src/core/model-pricing.ts`) — previously unpriced, so spend on these models read as invisible/free in our metering.

```bash
grep -n "sonnet-5\|fable-5\|claude-sonnet-5\|claude-fable-5" src/core/model-pricing.ts
gbrain search stats --days 7   # or whatever surfaces recent cost telemetry — confirm non-zero spend now attributed
```

---

## Self-review notes (from writing this plan)

- **Spec coverage:** all 3 fork features have a task (Jina/token-split → Task 4; wildcard → Task 5; `last_commit` → confirmed no-op, explicitly called out, no task needed). Merge mechanics (snapshot, conflict resolution, validation, landing) are Tasks 0/1/7/8/9/10/11.
- **Placeholder scan:** conflict-resolution steps can't show exact post-merge code because upstream's target shape isn't knowable until the merge actually runs — each such step instead gives the exact grep/diff commands to re-derive our side's intended behavior and an explicit list of what must survive, which is the strongest guarantee obtainable before the merge is actually performed.
- **Risk flagged explicitly:** `gateway.ts` and `operations.ts` (Tasks 4 and 5) are called out as the two files needing the most manual judgment; every other conflict is mechanical.
- **Adoption opportunities scoped separately (Task 12):** the CHANGELOG analysis surfaced several upstream features worth deliberately turning on (`--surface verbs`, `--bound-slug-prefixes`, the `apply-migrations` bugfix, pricing-table gaps) — these are investigation-only steps here, kept out of the merge commit itself so each can land as its own reviewable follow-up rather than being smuggled into a giant merge diff.
