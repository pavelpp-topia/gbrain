# Jina Recipe + Proper Token-Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace four env-var escape hatches with a proper Jina recipe + recipe-driven per-text truncation + config/env-backed concurrency.

**Architecture:** New `jina.ts` recipe declares `max_batch_tokens/chars_per_token/safety_factor`; `embed()` in gateway.ts computes the per-text char cap from those fields (general fix for all recipes); concurrency migrates from `_embedTuning.httpConcurrency` to `embed.http_concurrency` config key with `GBRAIN_EMBED_HTTP_CONCURRENCY` env override.

**Tech Stack:** TypeScript, Bun test, existing recipe pattern from `src/core/ai/recipes/voyage.ts`

**Spec:** `docs/superpowers/specs/2026-06-17-jina-recipe-token-split-design.md`

---

## File Map

| File | Action |
|---|---|
| `src/core/ai/recipes/jina.ts` | **Create** — Jina embedding recipe |
| `src/core/ai/recipes/index.ts` | **Modify** — register jina |
| `src/core/ai/build-gateway-config.ts` | **Modify** — thread `JINA_BASE_URL` + `embed.http_concurrency` |
| `src/core/ai/types.ts` | **Modify** — add `embed_http_concurrency?: number` to `AIGatewayConfig` |
| `src/core/ai/gateway.ts` | **Modify** — remove `_embedTuning`, generalize per-text truncation, wire `_embedHttpConcurrency` |
| `src/core/config.ts` | **Modify** — add `'embed.http_concurrency'` to `KNOWN_CONFIG_KEYS` |
| `test/ai/adaptive-embed-batch.test.ts` | **Modify** — rewrite section 8, add per-text truncation test, remove old seam |

---

## Task 1: Jina recipe + registration

**Files:**
- Create: `src/core/ai/recipes/jina.ts`
- Modify: `src/core/ai/recipes/index.ts`
- Modify: `src/core/ai/build-gateway-config.ts`

- [ ] **Step 1: Create `src/core/ai/recipes/jina.ts`**

```typescript
import type { Recipe } from '../types.ts';

/**
 * Jina Embeddings via a self-hosted text-embeddings-inference (TEI) server.
 *
 * TEI with jina-v2-base-code ONNX enforces a 2048-token per-request limit.
 * Code tokenizes at ~1 char/token; safety_factor=0.5 gives a 1024-char
 * per-text budget — well under the hard limit even with tokenizer variance.
 *
 * The model list below covers the most common Jina v2/v3 models. Because TEI
 * serves whatever model you launched it with, user_provided_models is set so
 * the gateway accepts any `jina:<id>` the user supplies.
 */
export const jina: Recipe = {
  id: 'jina',
  name: 'Jina Embeddings (self-hosted TEI)',
  tier: 'openai-compat',
  implementation: 'openai-compatible',
  base_url_default: 'http://localhost:8080/v1',
  auth_env: {
    required: [],
    optional: ['JINA_BASE_URL', 'JINA_API_KEY'],
    setup_url: 'https://huggingface.co/jinaai',
  },
  touchpoints: {
    embedding: {
      models: [
        'jina-embeddings-v2-base-code',
        'jina-embeddings-v2-base-en',
        'jina-embeddings-v2-small-en',
        'jina-embeddings-v3',
      ],
      user_provided_models: true,
      default_dims: 768,
      cost_per_1m_tokens_usd: 0,
      price_last_verified: '2026-06-17',
      max_batch_tokens: 2048,
      chars_per_token: 1,
      safety_factor: 0.5,
    },
  },
  setup_hint:
    'Run text-embeddings-inference with a Jina model. Set JINA_BASE_URL to the server URL.',
};
```

- [ ] **Step 2: Register jina in `src/core/ai/recipes/index.ts`**

Add the import after the `zeroentropyai` import (line ~19):
```typescript
import { jina } from './jina.ts';
```

Add to the `ALL` array after `zeroentropyai` (line ~38):
```typescript
  jina,
```

- [ ] **Step 3: Thread `JINA_BASE_URL` in `src/core/ai/build-gateway-config.ts`**

Add after the `OLLAMA_BASE_URL` block (~line 49):
```typescript
  if (process.env.JINA_BASE_URL) envBaseUrls['jina'] = process.env.JINA_BASE_URL;
```

- [ ] **Step 4: Typecheck**

```bash
bun run typecheck > /tmp/tc_task1.txt 2>&1; echo "EXIT=$?"; tail -5 /tmp/tc_task1.txt
```

Expected: `EXIT=0`

- [ ] **Step 5: Commit**

```bash
git add src/core/ai/recipes/jina.ts src/core/ai/recipes/index.ts src/core/ai/build-gateway-config.ts
git commit -m "feat(embed): add Jina Embeddings recipe for self-hosted TEI"
```

---

## Task 2: Generalize per-text truncation (TDD)

The current upstream `embed()` truncates all texts to `MAX_CHARS=8000` before splitting. When a recipe declares `max_batch_tokens`, individual texts can still exceed the per-batch budget even after splitting (a batch of 1 oversized text still hangs the provider). Fix: compute a tighter per-text cap from the recipe's token budget.

**Files:**
- Modify: `test/ai/adaptive-embed-batch.test.ts`
- Modify: `src/core/ai/gateway.ts`

- [ ] **Step 1: Write the failing test**

Add this new `describe` block at the end of `test/ai/adaptive-embed-batch.test.ts` (before the closing of the file, replacing the existing section 8 header comment with section 9):

```typescript
// --------- 9. Per-text truncation from recipe token budget ---------
//
// When a recipe declares max_batch_tokens, embed() must cap each individual
// text to (max_batch_tokens × safety_factor × chars_per_token) chars before
// splitting. Without this, a single oversized chunk lands in a sub-batch of 1
// and still hangs the provider — splitByTokenBudget operates at batch level,
// not within a text. This fires for ANY recipe with max_batch_tokens, not
// just Jina.

describe('per-text truncation from recipe token budget', () => {
  beforeEach(() => resetGateway());
  afterEach(() => __setEmbedTransportForTests(null));

  test('Jina recipe: texts longer than per-text budget are truncated before splitting', async () => {
    // Jina: max_batch_tokens=2048, chars_per_token=1, safety_factor=0.5
    // → perTextMaxChars = min(8000, floor(2048 * 0.5 * 1)) = 1024
    configureGateway({
      embedding_model: 'jina:jina-embeddings-v2-base-code',
      embedding_dimensions: 768,
      env: {},
    });

    const received: string[][] = [];
    const stub = mock(async ({ values }: { values: string[] }) => {
      received.push([...values]);
      return { embeddings: values.map(() => new Array(768).fill(0.1)) };
    });
    __setEmbedTransportForTests(stub as any);

    // 3 texts of 2000 chars each — all exceed the 1024-char per-text budget.
    const texts = ['a'.repeat(2000), 'b'.repeat(2000), 'c'.repeat(2000)];
    const result = await embed(texts);

    expect(result).toHaveLength(3);
    // Every text sent to the provider must be ≤ 1024 chars.
    const allTexts = received.flat();
    expect(allTexts).toHaveLength(3);
    for (const t of allTexts) {
      expect(t.length).toBeLessThanOrEqual(1024);
    }
  });

  test('recipe with no max_batch_tokens: texts are not truncated below MAX_CHARS', async () => {
    // OpenAI has no max_batch_tokens — fast path, no per-text budget cap.
    configureOpenAI();

    const received: string[][] = [];
    const stub = mock(async ({ values }: { values: string[] }) => {
      received.push([...values]);
      return { embeddings: values.map(() => new Array(1536).fill(0.1)) };
    });
    __setEmbedTransportForTests(stub as any);

    // 3 texts of 2000 chars — under MAX_CHARS=8000, so no truncation.
    const texts = ['a'.repeat(2000), 'b'.repeat(2000), 'c'.repeat(2000)];
    await embed(texts);

    const allTexts = received.flat();
    for (const t of allTexts) {
      expect(t.length).toBe(2000); // untouched
    }
  });
});
```

- [ ] **Step 2: Run the failing test**

```bash
bun test test/ai/adaptive-embed-batch.test.ts --test-name-pattern "per-text truncation" > /tmp/pertext_fail.txt 2>&1; echo "EXIT=$?"; tail -15 /tmp/pertext_fail.txt
```

Expected: `EXIT=1` — the first test should fail because `embed()` currently uses env-var-based truncation (which defaults to MAX_CHARS=8000 when no env var is set).

- [ ] **Step 3: Fix `embed()` in `src/core/ai/gateway.ts`**

In `embed()`, find the per-text truncation block (around line 1470). Replace the entire env-var-based block:

```typescript
  // topia: when a per-request token budget is declared via env override, cap
  // individual texts so no single input can exceed TEI's max_batch_tokens and
  // hang. The cap is computed from the budget with the effective safety factor
  // applied (same shrink-aware factor used for batch splitting), so tokenizer
  // variance doesn't push a truncated text over the budget on arrival.
  // GBRAIN_EMBED_MAX_CHARS_PER_TEXT can override this directly (no token math)
  // for operators who know the safe char ceiling empirically (e.g. "1024").
  let perTextMaxChars: number = MAX_CHARS;
  if (_embedTuning.maxCharsPerTextOverride !== undefined) {
    perTextMaxChars = _embedTuning.maxCharsPerTextOverride;
  } else if (_embedTuning.maxBatchTokensOverride !== undefined) {
    const sf = effectiveSafetyFactor(recipe); // shrink-aware; default 0.8
    perTextMaxChars = Math.min(
      MAX_CHARS,
      Math.floor(_embedTuning.maxBatchTokensOverride * sf * (_embedTuning.charsPerTokenOverride ?? DEFAULT_CHARS_PER_TOKEN)),
    );
  }
  const truncated = texts.map(t => (t ?? '').slice(0, perTextMaxChars));
```

With:

```typescript
  // When the recipe declares a token budget, cap each text to fit within it.
  // Without this, a single chunk exceeding max_batch_tokens lands in a
  // sub-batch of 1 and still hangs the provider — splitting cannot help at
  // the individual-text level. Fires for any recipe with max_batch_tokens.
  const embedding = recipe.touchpoints?.embedding;
  const maxBatchTokens = embedding?.max_batch_tokens;
  const charsPerToken = embedding?.chars_per_token ?? DEFAULT_CHARS_PER_TOKEN;
  const perTextMaxChars = maxBatchTokens !== undefined
    ? Math.min(MAX_CHARS, Math.floor(maxBatchTokens * effectiveSafetyFactor(recipe) * charsPerToken))
    : MAX_CHARS;
  const truncated = texts.map(t => (t ?? '').slice(0, perTextMaxChars));
```

Also remove the duplicate `embedding` / `maxBatchTokens` / `charsPerToken` declarations that appear later in the same function (around line 1513). Those lines become:

```typescript
  // Pre-split is gated on maxBatchTokens. Recipes without it (e.g. OpenAI)
  // ride the fast path: one embedMany call, no recursion safety net.
  const batches = maxBatchTokens
    ? splitByTokenBudget(truncated, Math.floor(maxBatchTokens * effectiveSafetyFactor(recipe)), charsPerToken)
    : [truncated];
```

(The `embedding`, `maxBatchTokens`, and `charsPerToken` consts are already declared above — remove the duplicate `const embedding = ...`, `const maxBatchTokens = ...`, `const charsPerToken = ...` lines that follow.)

- [ ] **Step 4: Run the tests — verify new tests pass, existing tests still pass**

```bash
bun test test/ai/adaptive-embed-batch.test.ts > /tmp/pertext_pass.txt 2>&1; echo "EXIT=$?"; tail -20 /tmp/pertext_pass.txt
```

Expected: `EXIT=0`, all tests pass.

- [ ] **Step 5: Typecheck**

```bash
bun run typecheck > /tmp/tc_task2.txt 2>&1; echo "EXIT=$?"; tail -5 /tmp/tc_task2.txt
```

Expected: `EXIT=0`

- [ ] **Step 6: Commit**

```bash
git add src/core/ai/gateway.ts test/ai/adaptive-embed-batch.test.ts
git commit -m "fix(embed): generalize per-text truncation to any recipe with max_batch_tokens"
```

---

## Task 3: Concurrency from config/env — remove `_embedTuning`

**Files:**
- Modify: `src/core/ai/types.ts`
- Modify: `src/core/ai/build-gateway-config.ts`
- Modify: `src/core/config.ts`
- Modify: `src/core/ai/gateway.ts`

- [ ] **Step 1: Add `embed_http_concurrency` to `AIGatewayConfig` in `src/core/ai/types.ts`**

In `AIGatewayConfig` (around line 339), add after the `env` field:

```typescript
  /**
   * Max concurrent embed sub-batch HTTP calls. Sourced from brain config
   * `embed.http_concurrency` (via buildGatewayConfig); `GBRAIN_EMBED_HTTP_CONCURRENCY`
   * env var overrides at configureGateway time. Default 1 (sequential).
   */
  embed_http_concurrency?: number;
```

- [ ] **Step 2: Thread `embed.http_concurrency` through `buildGatewayConfig`**

In `src/core/ai/build-gateway-config.ts`, add to the return object after `env`:

```typescript
    embed_http_concurrency: c.embed?.http_concurrency,
```

- [ ] **Step 3: Add `embed.http_concurrency` to `KNOWN_CONFIG_KEYS` in `src/core/config.ts`**

Find the `embed.backfill_*` block (around line 926) and add:

```typescript
  'embed.http_concurrency',
```

alongside the existing `embed.backfill_*` entries.

- [ ] **Step 4: Rework `gateway.ts` — remove `_embedTuning`, wire concurrency**

**4a.** Replace the entire `_embedTuning` block (lines ~80–144, everything from the comment `// Self-hosted-embedder throughput tuning` through the `releaseEmbedSlot` function) with:

```typescript
// ---------------------------------------------------------------------------
// Embed sub-batch concurrency.
//
// _embedHttpConcurrency bounds concurrent sub-batch HTTP calls across the
// whole process. Set via brain config embed.http_concurrency (normal path)
// or GBRAIN_EMBED_HTTP_CONCURRENCY env var (incident-time override; env wins).
// Default 1 preserves the prior sequential behaviour.
// ---------------------------------------------------------------------------
function resolveIntEnv(envVar: string): number | undefined {
  const raw = process.env[envVar];
  if (raw === undefined || raw === '') return undefined;
  const n = Number(raw);
  return Number.isFinite(n) && n > 0 ? Math.floor(n) : undefined;
}

let _embedHttpConcurrency = 1;

// Process-global FIFO semaphore bounding concurrent embed sub-batch HTTP
// calls at `_embedHttpConcurrency`. acquire() takes a slot or queues;
// release() hands the slot directly to the next waiter (count unchanged)
// or frees it. Fair FIFO; no busy-wait.
let _embedInFlight = 0;
const _embedSlotWaiters: Array<() => void> = [];
async function acquireEmbedSlot(): Promise<void> {
  if (_embedInFlight < _embedHttpConcurrency) {
    _embedInFlight++;
    return;
  }
  await new Promise<void>((resolve) => _embedSlotWaiters.push(resolve));
}
function releaseEmbedSlot(): void {
  const next = _embedSlotWaiters.shift();
  if (next) next();
  else _embedInFlight = Math.max(0, _embedInFlight - 1);
}
```

**4b.** In `configureGateway()` (line ~458), add concurrency resolution after `_config = { ... }`:

```typescript
  // env var wins (incident-time override); config key is normal path; default 1.
  _embedHttpConcurrency =
    resolveIntEnv('GBRAIN_EMBED_HTTP_CONCURRENCY') ??
    config.embed_http_concurrency ??
    1;
```

**4c.** In `resetGateway()`, replace:
```typescript
  _embedInFlight = 0;
  _embedSlotWaiters.length = 0;
```
with (no change needed — these lines stay as-is; just ensure `_embedHttpConcurrency` is also reset):
```typescript
  _embedHttpConcurrency = 1;
  _embedInFlight = 0;
  _embedSlotWaiters.length = 0;
```

**4d.** Replace `__setEmbedTuningForTests` (lines ~636–661) with:

```typescript
/**
 * Test-only seam for embed concurrency. Pass a number to set the limit;
 * pass null to restore the default (1). Resets semaphore state.
 * @internal
 */
export function __setEmbedConcurrencyForTests(n: number | null): void {
  _embedHttpConcurrency = n ?? 1;
  _embedInFlight = 0;
  _embedSlotWaiters.length = 0;
}
```

- [ ] **Step 5: Typecheck**

```bash
bun run typecheck > /tmp/tc_task3.txt 2>&1; echo "EXIT=$?"; tail -5 /tmp/tc_task3.txt
```

Expected: `EXIT=0`

- [ ] **Step 6: Commit**

```bash
git add src/core/ai/types.ts src/core/ai/build-gateway-config.ts src/core/config.ts src/core/ai/gateway.ts
git commit -m "feat(embed): embed.http_concurrency config key + remove _embedTuning env vars"
```

---

## Task 4: Rewrite section 8 tests

The existing section 8 tests used `__setEmbedTuningForTests` to inject env-var overrides for a recipe with no `max_batch_tokens`. Now that Jina has the right fields declared, the tests configure `jina:jina-embeddings-v2-base-code` directly and use `__setEmbedConcurrencyForTests` for the concurrency tests.

**Files:**
- Modify: `test/ai/adaptive-embed-batch.test.ts`

- [ ] **Step 1: Update imports**

Remove `__setEmbedTuningForTests` from the import line and add `__setEmbedConcurrencyForTests`:

```typescript
import {
  configureGateway,
  resetGateway,
  embed,
  splitByTokenBudget,
  isTokenLimitError,
  __setEmbedTransportForTests,
  __getShrinkStateForTests,
  __setEmbedConcurrencyForTests,
} from '../../src/core/ai/gateway.ts';
```

- [ ] **Step 2: Add `configureJina` helper**

Add after `configureGoogle()` (around line 55):

```typescript
function configureJina(): void {
  configureGateway({
    embedding_model: 'jina:jina-embeddings-v2-base-code',
    embedding_dimensions: 768,
    env: {},
  });
}
```

- [ ] **Step 3: Replace section 8 with rewritten tests using Jina recipe**

Replace the entire section 8 block (from `// --------- 8. Topia self-hosted TEI` through the end of the last `describe` in that section) with:

```typescript
// --------- 8. Jina / self-hosted TEI: recipe-driven token budget + concurrent dispatch ---------
//
// These tests cover the two production issues solved by the Jina recipe:
//
//   A. max_batch_tokens=2048 in the Jina recipe triggers pre-splitting so
//      pages with many chunks are never sent as one over-budget request.
//
//   B. embed.http_concurrency (set via __setEmbedConcurrencyForTests in tests)
//      fans out sub-batches concurrently, saturating multiple TEI replicas.
//      Output order is preserved via indexed Promise.all.

describe('Jina recipe: recipe-driven pre-splitting', () => {
  beforeEach(() => resetGateway());
  afterEach(() => __setEmbedTransportForTests(null));

  test('texts exceeding per-batch budget are split into multiple sub-batches', async () => {
    // Jina: max_batch_tokens=2048, chars_per_token=1, safety_factor=0.5
    // → batch budget = floor(2048 * 0.5) = 1024 chars
    // 4 texts of 700 chars each: 700 < 1024 so one fits, 700+700=1400 > 1024
    // → each goes into its own batch.
    configureJina();

    const stub = mock(async ({ values }: { values: string[] }) =>
      ({ embeddings: values.map(() => new Array(768).fill(0.1)) })
    );
    __setEmbedTransportForTests(stub as any);

    const texts = Array.from({ length: 4 }, () => 'x'.repeat(700));
    const result = await embed(texts);

    expect(result).toHaveLength(4);
    // 4 texts × 700 chars, budget 1024 → 4 sub-batches of 1.
    expect(stub).toHaveBeenCalledTimes(4);
  });

  test('texts fitting within budget are batched together', async () => {
    // 4 texts of 200 chars each: 200+200=400, 400+200=600, 600+200=800 < 1024
    // → all 4 fit in one batch.
    configureJina();

    const stub = mock(async ({ values }: { values: string[] }) =>
      ({ embeddings: values.map(() => new Array(768).fill(0.1)) })
    );
    __setEmbedTransportForTests(stub as any);

    const texts = Array.from({ length: 4 }, () => 'x'.repeat(200));
    const result = await embed(texts);

    expect(result).toHaveLength(4);
    expect(stub).toHaveBeenCalledTimes(1);
  });

  test('output order preserved across sub-batches', async () => {
    // 3 texts of 700 chars → 3 sub-batches. Each call's index encodes in
    // slot[0] so we can verify the final concat is in input order.
    configureJina();

    let callIdx = 0;
    const stub = mock(async ({ values }: { values: string[] }) => {
      const idx = callIdx++;
      return { embeddings: values.map(() => Array.from({ length: 768 }, (_, j) => j === 0 ? idx : 0)) };
    });
    __setEmbedTransportForTests(stub as any);

    const texts = ['a'.repeat(700), 'b'.repeat(700), 'c'.repeat(700)];
    const result = await embed(texts);

    expect(result).toHaveLength(3);
    expect(result.map(v => v[0])).toEqual([0, 1, 2]);
  });
});

describe('Jina recipe: concurrent sub-batch dispatch', () => {
  beforeEach(() => resetGateway());
  afterEach(() => {
    __setEmbedTransportForTests(null);
    __setEmbedConcurrencyForTests(null);
  });

  test('semaphore limits concurrent in-flight calls to the configured limit', async () => {
    configureJina();
    // 5 sub-batches but max 2 concurrent.
    __setEmbedConcurrencyForTests(2);

    let concurrent = 0;
    let maxSeen = 0;
    const stub = mock(async ({ values }: { values: string[] }) => {
      concurrent++;
      maxSeen = Math.max(maxSeen, concurrent);
      await new Promise(r => setTimeout(r, 0));
      concurrent--;
      return { embeddings: values.map(() => new Array(768).fill(0.1)) };
    });
    __setEmbedTransportForTests(stub as any);

    // 5 texts of 700 chars → 5 sub-batches of 1.
    const texts = Array.from({ length: 5 }, () => 'x'.repeat(700));
    await embed(texts);

    expect(maxSeen).toBeLessThanOrEqual(2);
    expect(stub).toHaveBeenCalledTimes(5);
  });

  test('concurrency=1 (default) executes sub-batches sequentially', async () => {
    configureJina();
    __setEmbedConcurrencyForTests(1);

    const callOrder: number[] = [];
    const stub = mock(async ({ values }: { values: string[] }) => {
      callOrder.push(callOrder.length);
      await new Promise(r => setTimeout(r, 0));
      return { embeddings: values.map(() => new Array(768).fill(0.1)) };
    });
    __setEmbedTransportForTests(stub as any);

    // 3 sub-batches of 1 text each.
    const texts = Array.from({ length: 3 }, () => 'x'.repeat(700));
    const result = await embed(texts);

    expect(stub).toHaveBeenCalledTimes(3);
    expect(result).toHaveLength(3);
    expect(callOrder).toEqual([0, 1, 2]);
  });
});
```

- [ ] **Step 4: Run the full adaptive-embed-batch test suite**

```bash
bun test test/ai/adaptive-embed-batch.test.ts > /tmp/embed_tests.txt 2>&1; echo "EXIT=$?"; grep -E "(✓|✗|fail|pass)" /tmp/embed_tests.txt | tail -30
```

Expected: `EXIT=0`, all tests pass (sections 1–9 inclusive).

- [ ] **Step 5: Commit**

```bash
git add test/ai/adaptive-embed-batch.test.ts
git commit -m "test(embed): rewrite section 8 to use Jina recipe + __setEmbedConcurrencyForTests"
```

---

## Task 5: Verification

- [ ] **Step 1: Run full unit test suite**

```bash
bun test > /tmp/full_tests.txt 2>&1; echo "EXIT=$?"; grep -E "fail|error" /tmp/full_tests.txt | grep -v "^#" | head -20; tail -5 /tmp/full_tests.txt
```

Expected: `EXIT=0`

- [ ] **Step 2: Run verify scripts**

```bash
bun run verify > /tmp/verify.txt 2>&1; echo "EXIT=$?"; tail -10 /tmp/verify.txt
```

Expected: `EXIT=0`

- [ ] **Step 3: Confirm warnRecipesMissingBatchTokens does NOT warn for Jina**

Jina declares `max_batch_tokens: 2048`, so the warning guard
(`if (!embedding || embedding.max_batch_tokens !== undefined) continue`) will skip it. Verify:

```bash
bun test test/ai/adaptive-embed-batch.test.ts --test-name-pattern "startup warning" > /tmp/warn_test.txt 2>&1; echo "EXIT=$?"; cat /tmp/warn_test.txt
```

Expected: `EXIT=0` — the warning test still passes (Google still warns; Jina is silent).

- [ ] **Step 4: Confirm old env vars are gone**

```bash
grep -r "GBRAIN_EMBED_MAX_BATCH_TOKENS\|GBRAIN_EMBED_CHARS_PER_TOKEN\|GBRAIN_EMBED_MAX_CHARS_PER_TEXT\|_embedTuning\|__setEmbedTuningForTests" src/ test/ --include="*.ts" | grep -v "\.md"
```

Expected: no output (all references removed).

- [ ] **Step 5: Confirm `GBRAIN_EMBED_HTTP_CONCURRENCY` still present in gateway.ts**

```bash
grep "GBRAIN_EMBED_HTTP_CONCURRENCY" /Users/pavel/IdeaProjects/gbrain/src/core/ai/gateway.ts
```

Expected: one line in `configureGateway` reading the env var override.
