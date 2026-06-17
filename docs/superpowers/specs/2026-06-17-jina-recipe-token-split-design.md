# Jina Recipe + Proper Token-Split Design

**Date:** 2026-06-17
**Branch:** topia-main
**Replaces:** env-var commits `b862d64b`, `aa05bb7e`, `ab9f5c7a`

## Problem

The three env-var commits on `topia-main` solved a real production problem — self-hosted
TEI with jina-v2-base-code was hanging (~40% page error rate) because:

1. The `openai-compatible` implementation has no `max_batch_tokens` in the recipe, so
   the pre-split path never fires — all chunks go as one over-budget request.
2. A single chunk exceeding the token limit lands in a sub-batch of 1 and still hangs
   the provider — `splitByTokenBudget` operates at the batch level, not within a text.
3. With a single serial dispatch loop, 5 of 6 TEI replicas sit idle.

The fix worked, but used four process-level env vars (`GBRAIN_EMBED_MAX_BATCH_TOKENS`,
`GBRAIN_EMBED_CHARS_PER_TOKEN`, `GBRAIN_EMBED_MAX_CHARS_PER_TEXT`,
`GBRAIN_EMBED_HTTP_CONCURRENCY`) rather than following the upstream convention of
declaring provider characteristics in recipe config.

## Solution (Approach A)

### Core insight

`max_batch_tokens * safety_factor * chars_per_token` is the same formula the pre-split
path already uses for the batch budget — extending it one level down to the per-text cap
is the natural fix, not a new escape hatch. With the right recipe fields declared, all
three token-tuning env vars become unnecessary.

Concurrency is a deployment concern (depends on replica count), not a model property —
it belongs in config/env, not the recipe.

## Design

### 1. New file: `src/core/ai/recipes/jina.ts`

```typescript
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
      // TEI with jina-v2-base-code ONNX enforces a 2048-token per-request
      // limit. Code tokenizes ~1 char/token; 0.5 safety factor → 1024 char
      // per-text budget, well under the hard limit.
      max_batch_tokens: 2048,
      chars_per_token: 1,
      safety_factor: 0.5,
    },
  },
  setup_hint:
    'Run text-embeddings-inference with a Jina model. Set JINA_BASE_URL to the server URL.',
};
```

Register in `src/core/ai/recipes/index.ts`.

The three token-tuning env vars (`GBRAIN_EMBED_MAX_BATCH_TOKENS`,
`GBRAIN_EMBED_CHARS_PER_TOKEN`, `GBRAIN_EMBED_MAX_CHARS_PER_TEXT`) are removed — the
Jina recipe arithmetically produces the same 1024-char per-text budget.

### 2. `src/core/ai/gateway.ts` — three targeted changes

**2a. Remove `_embedTuning` struct**

Delete: `_embedTuning` object, `resolveIntEnv` (if only used there),
`__setEmbedTuningForTests`, and all reads of the four env vars.

Keep: the semaphore (`_embedInFlight`, `_embedSlotWaiters`, `acquireEmbedSlot`,
`releaseEmbedSlot`) — fed by `_embedHttpConcurrency` (a plain `let`).

**2b. Generalize per-text truncation in `embed()`**

```typescript
// When the recipe declares a token budget, cap each text to fit within it.
// Without this, a single chunk exceeding max_batch_tokens lands in a
// sub-batch of 1 and still hangs the provider — splitting cannot help at
// the individual-text level.
const perTextMaxChars = maxBatchTokens !== undefined
  ? Math.min(MAX_CHARS, Math.floor(maxBatchTokens * effectiveSafetyFactor(recipe) * charsPerToken))
  : MAX_CHARS;
const truncated = texts.map(t => (t ?? '').slice(0, perTextMaxChars));
```

Fires for any recipe with `max_batch_tokens` (Voyage, ZE, Jina) — genuine upstream bug
fix, not Jina-specific.

`maxBatchTokens` and `charsPerToken` come from the recipe only — no env-var fallback:

```typescript
const maxBatchTokens = embedding?.max_batch_tokens;
const charsPerToken = embedding?.chars_per_token ?? DEFAULT_CHARS_PER_TOKEN;
```

**2c. Concurrency reads `_embedHttpConcurrency`**

No logic change to semaphore or dispatch loop. Resolution in `configureGateway()`:

```typescript
const envConcurrency = resolveIntEnv('GBRAIN_EMBED_HTTP_CONCURRENCY');
const cfgConcurrency = typeof cfg.embed?.http_concurrency === 'number'
  ? cfg.embed.http_concurrency : undefined;
_embedHttpConcurrency = envConcurrency ?? cfgConcurrency ?? 1;
```

Default 1 — sequential behavior preserved for everyone on upgrade.

### 3. `src/core/config.ts`

Add `'embed.http_concurrency'` to `KNOWN_CONFIG_KEYS` alongside `embed.backfill_*`.

### 4. `test/ai/adaptive-embed-batch.test.ts`

- Replace `__setEmbedTuningForTests({maxBatchTokensOverride, ...})` with recipes that
  declare the equivalent fields (`max_batch_tokens`, `chars_per_token`, `safety_factor`)
- Replace concurrency injection with `__setEmbedConcurrencyForTests(n: number | null)`
- All 17 existing test cases preserved
- Add one new case: per-text truncation fires for a Voyage-shaped recipe (not just Jina),
  confirming it is a general fix

## What is removed

| Removed | Replaced by |
|---|---|
| `GBRAIN_EMBED_MAX_BATCH_TOKENS` | `jina.touchpoints.embedding.max_batch_tokens` |
| `GBRAIN_EMBED_CHARS_PER_TOKEN` | `jina.touchpoints.embedding.chars_per_token` |
| `GBRAIN_EMBED_MAX_CHARS_PER_TEXT` | computed from recipe: `max_batch_tokens × safety_factor × chars_per_token` |
| `_embedTuning` struct | `_embedHttpConcurrency` let + `configureGateway` wiring |
| `__setEmbedTuningForTests` | `__setEmbedConcurrencyForTests` (narrower seam) |

`GBRAIN_EMBED_HTTP_CONCURRENCY` env var is **kept** as an incident-time override,
subordinate to `embed.http_concurrency` config key.

## Files changed

| File | Change |
|---|---|
| `src/core/ai/recipes/jina.ts` | New |
| `src/core/ai/recipes/index.ts` | +1 line (register jina) |
| `src/core/config.ts` | +1 line (`embed.http_concurrency` to KNOWN_CONFIG_KEYS) |
| `src/core/ai/gateway.ts` | ~40 lines removed, ~15 changed |
| `test/ai/adaptive-embed-batch.test.ts` | Restructured in place, +1 test case |

## Upstream contribution path

- `jina.ts` recipe is self-contained and PR-worthy
- Per-text truncation fix (`2b`) applies to all providers — PR-worthy as a standalone fix
- Concurrent dispatch + `embed.http_concurrency` is topia-specific (TEI multi-replica),
  but clean enough to propose upstream as a general capability
