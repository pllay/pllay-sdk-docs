# Rate limits

## Limits, per partner key, per route

| Route shape | Limit |
|---|---|
| Any `GET` (`interactions.get()`/`.list()`, `outcomes.get()`, `campaigns.get()`, `analytics.getCampaignSummary()`, `intelligence.*`) | 120/min |
| `PATCH /v1/webhooks` | 10/min |
| Everything else (`interactions.create()`/`.close()`, `outcomes.submit()`) | 30/min |

These are enforced server-side, per `X-Partner-Key`, independently per route — exhausting your budget on `interactions.create()` doesn't affect your `campaigns.get()` calls. There is currently no way to raise these limits from the SDK side; contact your PLLAY partner engineering contact if your integration's real traffic needs a higher ceiling.

## What happens when you're limited

The API returns `429` with a `Retry-After` header (seconds). The SDK surfaces this as `PLLAYRateLimitError`:

```ts
import { PLLAYRateLimitError } from "@pllay/sdk";

try {
  await pllay.interactions.create({ /* ... */ });
} catch (err) {
  if (err instanceof PLLAYRateLimitError) {
    console.error(`Rate limited — retry after ${err.retryAfterSeconds}s`);
  }
}
```

## How the SDK's own retries interact with this

You don't need to hand-roll backoff for transient rate limiting — the SDK already does the right thing, within the bounds of its [Client SDKs aren't published yet. Integrate today with the HTTP API and sandbox.](https://www.pllay.io/developers):

- A **safe** call (any `GET`) that hits a `429` is retried automatically, honoring the server's `Retry-After` exactly (not the SDK's own jittered backoff — the server told you how long to wait, so that's what it waits).
- An **unsafe** call (`create()`, `close()`, `submit()`) that hits a `429` is retried **only if you passed an `idempotencyKey`** — same rule as every other transient failure. Without one, it throws `PLLAYRateLimitError` immediately on the first `429`, exactly once, with no retry.
- Retries stop after `maxRetries` (default `2`, i.e. 3 total attempts) — configurable via `new PLLAYClient({ maxRetries: N })`. If every attempt is rate-limited, the final `PLLAYRateLimitError` is what you catch, with `retryAfterSeconds` from the last response.
- Every retry (rate-limit or otherwise) respects your own `timeoutMs`/`AbortSignal` — a long `Retry-After` combined with a short per-call timeout will surface a `PLLAYAbortError` instead of waiting past your budget. If you expect to be rate-limited during a burst, pass a longer `timeoutMs` for that call rather than relying on the default 10s.

## Designing around the limits

- **Batch reads where the API supports it** rather than polling `interactions.get()` in a tight loop — `interactions.list()` (currently `@beta`) returns multiple interactions per call.
- **Always pass an `idempotencyKey` on mutations you might need to retry** — beyond making retries safe in general, it's what lets the SDK absorb a `429` on `create()`/`close()`/`submit()` instead of surfacing it to your own code immediately.
- **Treat `PLLAYRateLimitError` as expected, not exceptional**, in any code path that fires interactions/outcomes in bulk (e.g. backfilling historical data, bulk-closing at end of day) — catch it specifically and back off at the application level using `retryAfterSeconds`, rather than letting a generic error handler log-and-abandon the whole batch.
