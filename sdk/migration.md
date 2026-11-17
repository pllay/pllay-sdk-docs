# Migration guide

## From raw HTTP calls to the SDK

If you're currently calling the Partner API directly with `fetch`/`axios`/`node-fetch`, here's the direct mapping. Every difference below is either a safety improvement (retries, idempotency, typed errors) or a naming change (`prediction` → `interaction`, matching this SDK's public vocabulary — see the note in `src/types.ts` for why).

| Raw HTTP | SDK equivalent |
|---|---|
| `POST /v1/predictions/launch` with `X-Partner-Key` header | `pllay.interactions.create(input, { idempotencyKey })` |
| `GET /v1/predictions/{id}/state` | `pllay.interactions.get(id)` |
| `PATCH /v1/predictions/{id}/close` | `pllay.interactions.close(id, { idempotencyKey })` |
| `POST /v1/predictions/{id}/outcome` | `pllay.outcomes.submit(id, input, { idempotencyKey })` |
| `GET /v1/predictions/{id}/resolution` | `pllay.outcomes.get(id)` |
| `GET /v1/campaign-config` | `pllay.campaigns.get()` |
| `GET /v1/analytics` | `pllay.analytics.getCampaignSummary()` |
| Manually verifying `X-Pllay-Signature` with `crypto.createHmac` | `pllay.webhooks.verify()` / `.constructEvent()` |
| Hand-rolled retry-on-5xx loop | Built in — safe by default, see [Client SDKs aren't published yet. Integrate today with the HTTP API and sandbox.](https://www.pllay.io/developers) |
| Manually generating and tracking your own idempotency keys | Same concept, just pass `{ idempotencyKey }` — the SDK doesn't invent this, it exposes exactly what the API already supports |

### Before

```ts
const res = await fetch("https://.../functions/v1/partner-api/v1/predictions/launch", {
  method: "POST",
  headers: { "X-Partner-Key": apiKey, "Content-Type": "application/json" },
  body: JSON.stringify({ creator_external_id: "creator-123", question: "...", options: ["Yes", "No"] }),
});
if (!res.ok) {
  const body = await res.json();
  throw new Error(body.error?.message ?? `HTTP ${res.status}`);
}
const data = await res.json();
```

### After

```ts
const interaction = await pllay.interactions.create({
  creatorExternalId: "creator-123",
  question: "...",
  options: ["Yes", "No"],
});
```

Error handling collapses from manually checking `res.ok` and parsing the error envelope to catching typed error classes — see [Client SDKs aren't published yet. Integrate today with the HTTP API and sandbox.](https://www.pllay.io/developers).

## SDK version migrations (future)

This SDK follows [Semantic Versioning](https://semver.org/):

- **Patch** (`0.1.x`): bug fixes, no API changes.
- **Minor** (`0.x.0`): additive changes only — a new method, a new optional field, a new error subclass. Existing code keeps working.
- **Major** (`x.0.0`): breaking changes. A migration section will be added here for each one, with a before/after diff like the one above.

`interactions.list()` is currently marked **preview** — its shape may change in a minor release without that counting as a breaking change (this is called out explicitly wherever the method is documented). Everything else in `0.1.0` is considered stable from release.

See [`CHANGELOG.md`](../CHANGELOG.md) for the version-by-version record.
