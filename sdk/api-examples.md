# API examples

All examples assume:

```ts
import { PLLAYClient } from "@pllay/sdk";
const pllay = new PLLAYClient({ apiKey: process.env.PLLAY_API_KEY! });
```

## interactions.create()

```ts
const interaction = await pllay.interactions.create(
  {
    creatorExternalId: "creator-123",
    question: "Will the home team win?",
    options: ["Yes", "No"],   // 2-4 required
    timerSeconds: 60,          // 15-600, defaults to 60
  },
  { idempotencyKey: crypto.randomUUID() }, // recommended
);
// => { interactionId, interactionUrl, embedUrl }
```

Throws `PLLAYValidationError` (client-side, no network call) for bad input, or `PLLAYNotFoundError` (code `creator_not_linked`) if `creatorExternalId` isn't linked to your account.

## interactions.get()

```ts
const state = await pllay.interactions.get(interaction.interactionId);
// => { interactionId, status, poolTotalCents, options, odds, timeRemainingSeconds }
```

Read-only, always safe to retry. Throws `PLLAYNotFoundError` if the interaction doesn't exist — including if it exists but belongs to a different partner (deliberately indistinguishable, to prevent enumeration of other partners' interaction IDs).

## interactions.close()

```ts
await pllay.interactions.close(interaction.interactionId, { idempotencyKey: `close-${interaction.interactionId}` });
// => { interactionId, status: "LOCKED" }
```

Stops new stake. Only valid from `OPEN` or `COUNTDOWN` — throws `PLLAYConflictError` otherwise.

## interactions.list() *(preview)*

```ts
const page = await pllay.interactions.list({ limit: 25 });
// => { data: Interaction[], pagination: { nextCursor, hasMore } }

if (page.pagination.hasMore) {
  const nextPage = await pllay.interactions.list({ limit: 25, cursor: page.pagination.nextCursor });
}
```

> **Preview**: this method is additive to the originally-scoped surface. Its pagination shape may be refined in a future minor release.

## outcomes.submit()

```ts
const outcome = await pllay.outcomes.submit(
  interaction.interactionId,
  { winningOutcome: "Yes" },              // or { winningOptionIndex: 0 }
  { idempotencyKey: `outcome-${interaction.interactionId}` },
);
// => { interactionId, winningOptionIndex, winner, serviceFeeCents, partnerRevenueCents, pllayRevenueCents }
```

**Settles the interaction immediately and authoritatively.** There is no separate propose/confirm step — submit only when you're certain.

## outcomes.get()

```ts
const resolution = await pllay.outcomes.get(interaction.interactionId);
// => { interactionId, status, winningOptionIndex, winner, resolutionSource, resolvedBy, resolvedAt }
```

Throws `PLLAYConflictError` (code `not_resolved`) if the interaction hasn't settled yet.

## campaigns.get()

```ts
const config = await pllay.campaigns.get();
// => { partnerId, takeRate, partnerTake, brandConfig, model: "fixed_share" }
```

Your partner account's own commercial terms — not a Pulse-style sponsor campaign object. Partner-attributed interactions always use the standard fixed-share model.

## analytics.getCampaignSummary()

```ts
const summary = await pllay.analytics.getCampaignSummary();
// => { partnerId, thisMonthCents, lastMonthCents, totalCents, pendingPayoutCents, interactionCount }
```

A partner-wide summary (there is currently only one "campaign" — your whole account).

## intelligence.getSummary()

```ts
const summary = await pllay.intelligence.getSummary({ windowDays: 28 });
// => { partnerId, dataLabeling, windowDays, contentSourcesInScope, kFloor, suppressed,
//      participants, repeatParticipants, interactionsPerParticipant, sessionRetentionPct }
```

Only usable by a partner account provisioned for the Platform Intelligence product (`product_domain = 'intelligence'`) — a predictions-hosting partner key gets a 403. `windowDays` is optional (1-90, default 28). k-anonymity: when `suppressed` is `true` (fewer than `kFloor` participants in the window, matching the server's `graph_k_floor()`), `participants`/`repeatParticipants`/`interactionsPerParticipant`/`sessionRetentionPct` are all `null` — not missing data, a declined-to-identify small cohort. See `docs/PLATFORM_INTELLIGENCE_SPEC.md` in the main repo.

## intelligence.getMoments()

```ts
const moments = await pllay.intelligence.getMoments({ windowDays: 28, limit: 10 });
// => { partnerId, dataLabeling, windowDays, momentsInScope, kFloor,
//      moments: [{ eventId, eventType, suppressed, participants, returnedParticipants, returnRate, revenueCents }, ...] }
```

Same k-anonymity rule as `getSummary()`, applied per moment: a moment with `suppressed: true` had fewer than `kFloor` participants, so its numeric fields are `null` while `eventId`/`eventType` still identify which of your events it was.

Your own content's moments, ranked by `returnRate` descending. Same partner
gating as `getSummary()`. `limit` is optional (1-100, default 25). Does not
identify a creator — only `eventId`/`eventType`-level aggregates.

## intelligence.getAudience()

```ts
const audience = await pllay.intelligence.getAudience({ limit: 10 });
// => { partnerId, dataLabeling, kFloor, cohortsInScope,
//      audience: [{ campaignId, contentCategory, suppressed, cohortSize, meanAccuracy, meanEventsPerFan }, ...] }
```

Your own audience composition, k-anonymized by content category. Same
partner gating as `getSummary()`. No `windowDays` — this is a lifetime
cohort aggregate per campaign × content category, not a rolling window.
Same k-anonymity rule as the other two methods, applied per cohort: a
cohort with `suppressed: true` had fewer than `kFloor` fans, so
`cohortSize`/`meanAccuracy`/`meanEventsPerFan` are `null` while
`campaignId`/`contentCategory` still identify which cohort it was.
`limit` is optional (1-100, default 25), ranked non-suppressed cohorts
first, then by `cohortSize` descending.

## webhooks.verify() / .constructEvent()

```ts
import { PLLAYClient } from "@pllay/sdk"; // Node.js only

app.post("/webhooks/pllay", express.raw({ type: "application/json" }), (req, res) => {
  try {
    const event = pllay.webhooks.constructEvent(
      req.body.toString("utf8"),          // the RAW body — not parsed JSON
      req.header("X-Pllay-Signature")!,
      process.env.PLLAY_WEBHOOK_SECRET!,
    );
    if (event.event === "prediction.settled") {
      // handle settlement
    }
    res.sendStatus(200);
  } catch {
    res.sendStatus(400); // invalid signature — reject
  }
});
```

See [`examples/webhook-node`](../examples/webhook-node) for the full, runnable example.

## Cancellation with AbortSignal

```ts
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000);

try {
  await pllay.interactions.get(id, { signal: controller.signal });
} catch (err) {
  if (err instanceof PLLAYAbortError) console.log("Cancelled or timed out");
}
```

## Custom timeout and retry count

```ts
const pllay = new PLLAYClient({
  apiKey: process.env.PLLAY_API_KEY!,
  timeoutMs: 5000,   // default 10000
  maxRetries: 1,      // default 2 — applies only to safe reads and idempotent mutations
});
```
