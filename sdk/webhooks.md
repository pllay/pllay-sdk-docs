# Webhooks

## What you receive

Two event types today, delivered as an HTTP `POST` to the URL you configure:

| Event | Fires when |
|---|---|
| `prediction.started` | An interaction you launched transitions to `OPEN` |
| `prediction.settled` | An interaction you launched settles |

```json
{
  "event": "prediction.settled",
  "event_id": "3f8e2b10-...",
  "attempt": 1,
  "sent_at": "2026-08-31T00:00:00.000Z",
  "data": { "...event-specific fields..." }
}
```

`event_id` identifies the underlying interaction, not the delivery — it repeats across every retry attempt of the same delivery and across the different event types fired for the same interaction. To de-duplicate a specific delivery attempt, use the `X-Pllay-Delivery` header instead (unique per `(event_id, event_type, attempt)` triple).

## Setting your webhook URL

```ts
// Currently a raw HTTP call — this SDK does not yet wrap PATCH /v1/webhooks
// as a method (see the "Not yet in the SDK" note below).
const res = await fetch(`${baseUrl}/v1/webhooks`, {
  method: "PATCH",
  headers: { "X-Partner-Key": apiKey, "Content-Type": "application/json" },
  body: JSON.stringify({ webhook_url: "https://your-app.example.com/webhooks/pllay" }),
});
const { webhook_secret } = await res.json();
// Store webhook_secret now — it is returned exactly once and never shown again.
```

Every time you set or change your `webhook_url`, the signing secret **rotates** — the new one is returned in that same response, once. If you lose it, set the URL again (even to the same value) to get a fresh one; there's no way to retrieve a previously-issued secret.

## Verifying a delivery

Use `webhooks.verify()` or `webhooks.constructEvent()` from the Node.js entry point — **never** available from `@pllay/sdk/browser`, since verification requires your server-only `webhook_secret`:

```ts
import { PLLAYClient, PLLAYAuthError } from "@pllay/sdk";
import express from "express";

const pllay = new PLLAYClient({ apiKey: process.env.PLLAY_API_KEY! });
const app = express();

// IMPORTANT: verification is byte-exact over the raw request body.
// Do NOT use express.json() on this route — it re-serializes the body,
// which will not reproduce the exact bytes PLLAY signed. Use a raw parser
// scoped to this one route instead.
app.post("/webhooks/pllay", express.raw({ type: "application/json" }), (req, res) => {
  try {
    const event = pllay.webhooks.constructEvent(
      req.body.toString("utf8"),
      req.header("X-Pllay-Signature") ?? "",
      process.env.PLLAY_WEBHOOK_SECRET!,
    );

    switch (event.event) {
      case "prediction.settled":
        // handle event.data
        break;
      case "prediction.started":
        // handle event.data
        break;
    }

    res.sendStatus(200); // ack quickly — see "Responding" below
  } catch (err) {
    if (err instanceof PLLAYAuthError) {
      res.sendStatus(400); // bad signature or malformed payload — do not retry-loop this
      return;
    }
    throw err;
  }
});
```

`constructEvent()` verifies the signature (constant-time comparison — see the module's own doc comment in `src/modules/webhooks.ts`) and parses the payload in one call, throwing `PLLAYAuthError` (`invalid_webhook_signature` or `invalid_webhook_payload`) if either check fails. Use the lower-level `webhooks.verify(rawBody, signatureHeader, secret): boolean` directly if you want to check the signature without also requiring the body to parse as PLLAY's exact envelope shape.

See [`examples/webhook-node`](../examples/webhook-node) for a complete, runnable handler, including how to compute a valid test signature locally without a live webhook delivery.

## Responding

Return `2xx` as soon as you've durably queued the event for your own processing — don't do slow work (external API calls, heavy computation) synchronously in the handler before responding. PLLAY's delivery attempt has an 8-second timeout; a slow handler causes a spurious retry of an event you already received.

## Delivery, retries, and idempotency

This is entirely server-side behavior — nothing here is configurable from the SDK:

- Up to 3 attempts per event, backoff `0s / 4s / 16s` between attempts.
- After 3 failed attempts, the delivery is marked `dead` — there is currently no automatic redelivery.
- **Your handler must be idempotent.** A delivery can arrive more than once for the same logical event — a retry after your `2xx` response was lost in transit, for instance. Key your own side-effects off `event_id` (or `X-Pllay-Delivery` for per-attempt granularity) and make re-processing the same event a safe no-op.

See [`docs/PARTNER_API.md`'s Webhooks section](../../../docs/PARTNER_API.md#webhooks) in the main repo for the full server-side mechanics (this is the authoritative source; this page is the SDK-consumer's view of the same system).

## Not yet in the SDK

- `PATCH /v1/webhooks` (setting/rotating your URL) isn't wrapped as an SDK method yet — call it directly as shown above.
- There is a second, newer, separate webhook system in development for a different event taxonomy (`interaction.*`/`reward.*`/`campaign.*`, Pulse-side) with its own signing scheme (`X-Pllay-Signature: t=<timestamp>,v1=<hex>`, timestamp-bound for replay protection). It is not partner-facing yet and `webhooks.verify()`/`.constructEvent()` above do **not** understand its signature format — this doc will be updated when that changes.
