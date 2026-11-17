# Going to production

This is the **partner's** checklist for moving an integration from sandbox to live traffic. It's a different document from [`RELEASE_PROCESS.md`](RELEASE_PROCESS.md), which covers how *this SDK itself* gets published — read that one only if you're contributing to `@pllay/sdk`, not integrating it.

## Before you start

**Confirm production availability directly with your PLLAY partner engineering contact before changing `environment`.** Which parts of the Partner API are enabled for production traffic can change independently of this SDK's own version — [`sandbox.md`](sandbox.md) is the source of truth for the current state as of when it was last updated, but treat "is my integration actually cleared for live traffic" as a conversation to have, not an assumption to make from documentation alone.

## Credentials

1. **Request a `pllay_sk_live_...` key**, scoped to exactly the routes your integration uses (see the scopes table in [authentication.md](authentication.md)) — don't request broader scopes than you need.
2. **Store it the same way you'd store any production secret**: your platform's secret manager or environment variables, never in source control, never logged. This SDK never logs the key itself, but audit your own logging/error-reporting setup (Sentry, Datadog, etc.) to make sure a thrown error's context doesn't accidentally capture request headers.
3. **Rotation plan**: there is currently no in-place key rotation — a new key is a new `issue_partner_api_key` call, and the old one is separately revoked. If your on-call process assumes zero-downtime credential rotation, plan for a brief dual-key window (old key still valid while the new one is deployed) rather than a hard cutover, and coordinate the old key's revocation with PLLAY once you've confirmed the new one is live everywhere it needs to be.
4. **Webhook secret**: if you use webhooks, setting your production `webhook_url` (see [webhooks.md](webhooks.md)) mints a **separate** secret from whatever you used in sandbox — update it in your production config, not just the URL.

## Configuration changes from sandbox

```ts
const pllay = new PLLAYClient({
  apiKey: process.env.PLLAY_API_KEY!,   // your NEW live key — sandbox and live keys are not interchangeable
  environment: "production",             // was "sandbox"
});
```

Nothing else about the SDK's API surface changes between environments — the same methods, same types, same error classes. If your integration behaves differently against production than it did in sandbox, that's a signal to investigate (different data shape, different rate limit headroom, a scope you have in one environment but not the other), not something to work around client-side.

## Operational readiness

- **Idempotency keys on every mutation your own retry logic might repeat.** In sandbox this is easy to skip and still "work" (low traffic, few real failures to retry). In production, a network blip during `interactions.create()` without an idempotency key means a genuine duplicate interaction, not just a retried one — see the [Client SDKs aren't published yet. Integrate today with the HTTP API and sandbox.](https://www.pllay.io/developers).
- **Error handling coverage for every `PLLAYError` subclass you might realistically hit**, not just the happy path — at minimum `PLLAYRateLimitError` (see [rate-limits.md](rate-limits.md)), `PLLAYValidationError` (surface `fieldErrors` somewhere you'll see them, don't just log-and-swallow), and `PLLAYAuthError` (a production key can be revoked or hit a scope boundary you didn't exercise in sandbox testing).
- **Timeouts sized for your actual infrastructure.** The SDK's default `timeoutMs` (10s) is a reasonable default, not a guarantee tuned to your deployment — if you're calling PLLAY from a serverless function with its own tighter platform timeout, set `timeoutMs` (and pass your own `AbortSignal` if you need to propagate cancellation from an upstream request) so a slow PLLAY response fails predictably inside your own timeout budget, not after it.
- **Webhook handler idempotency**, if you use webhooks — see [webhooks.md](webhooks.md)'s note on this; it's easy to not notice a gap here in sandbox where delivery volume is low.
- **Monitoring**: at minimum, alert on a sustained rise in `PLLAYServerError`/`PLLAYNetworkError` rates (signals a PLLAY-side or connectivity issue, not your bug) separately from a rise in `PLLAYValidationError` (signals something in your own request-building logic drifted). Every `PLLAYError` carries `requestId`/`correlationId` — include these in whatever you log, so a support request to PLLAY can be traced to the exact call.

## First production call

Verify connectivity with a cheap, safe read before routing real user traffic through the integration:

```ts
const pllay = new PLLAYClient({ apiKey: process.env.PLLAY_API_KEY!, environment: "production" });
const config = await pllay.campaigns.get();
console.log("Connected to production as partner:", config.partnerId);
```

If this throws, resolve it before anything else — every other method depends on the same auth path.

## Rollback

If you need to revert to sandbox (e.g. a production-only issue surfaces): swap `environment` and `apiKey` back. Because sandbox and production are fully separate environments with independent data, this is a safe, immediate action — it does not affect or roll back anything already committed on the production side (settled interactions, delivered webhooks, etc. stay as they are).
