# Authentication

## Key types

| Prefix | Where it works | Scopes | Notes |
|---|---|---|---|
| `pllay_sk_live_...` | production | explicit, enforced | Issued via the `issue_partner_api_key` RPC (currently admin-only — ask PLLAY to provision one). |
| `pllay_sk_sandbox_...` | sandbox | explicit, enforced | Self-service — create, rotate, and revoke your own sandbox keys from the Developer Portal's Credentials page (`apps/developer-portal`), no PLLAY admin required. This is the **only key type you can self-test against today** — see [Sandbox guide](sandbox.md). |
| `pllay_pk_...` | production only | none — full access | The legacy key format. Works everywhere a scoped key does, since it predates scopes and is treated as carrying all of them. |

Scopes gate specific SDK methods:

| Scope | Gates |
|---|---|
| `interactions:read` | `interactions.get()`, `.list()` |
| `interactions:write` | `interactions.create()`, `.close()` |
| `outcomes:read` | `outcomes.get()` |
| `outcomes:write` | `outcomes.submit()` |
| `analytics:read` | `analytics.getCampaignSummary()` |
| `campaigns:read` | `campaigns.get()` |

A key lacking a required scope produces the same `PLLAYAuthError` as an invalid key — the SDK (and the underlying API) never reveals which specific scope a caller is missing, to avoid leaking information useful for probing a stolen key.

## Server-side use

```ts
import { PLLAYClient } from "@pllay/sdk";

const pllay = new PLLAYClient({ apiKey: process.env.PLLAY_API_KEY! });
```

Keep `apiKey` in an environment variable or your secret manager — never in source code, never in a client-visible config, never logged. This SDK never logs the key itself (it appears only as an outgoing request header), but your own code should apply the same discipline.

## Browser use

**PLLAY's Partner API does not currently issue a public or embeddable read token.** Every key it recognizes — legacy or scoped — is a full partner secret meant to live server-side only. Given that, `@pllay/sdk/browser`'s `PLLAYBrowserClient` is designed so that it is **structurally impossible** to misuse it into leaking a secret:

```ts
export interface PLLAYBrowserClientConfig {
  baseUrl: string;      // YOUR OWN backend — never PLLAY's API
  headers?: Record<string, string>; // your own session auth, never a PLLAY key
  // ... no apiKey field exists anywhere in this type.
}
```

There is no `apiKey` field to fill in, correctly or incorrectly. `PLLAYBrowserClient` sends requests to `${baseUrl}/v1/...` — your own backend, not PLLAY's — and your backend is responsible for authenticating the browser request however you already do (session cookie, your own bearer token, etc.), then proxying to PLLAY with `PLLAYClient` server-side.

### The proxy pattern

```
Browser (PLLAYBrowserClient)
   │  fetch("https://your-app.com/api/pllay/v1/predictions/launch", {...})
   ▼
Your backend (Next.js API route / Express route)
   │  validates the browser's OWN session
   │  calls pllay.interactions.create(...) with YOUR secret key, server-side
   ▼
PLLAY's API
```

`examples/nextjs` has a complete, working proxy implementation using this exact pattern — copy it as a starting point. The route shapes `PLLAYBrowserClient` calls match PLLAY's own `/v1/*` routes exactly (`POST /v1/predictions/launch`, `GET /v1/predictions/{id}/state`, etc.), so a passthrough proxy is a handful of lines per route.

```ts
// Browser code — this file NEVER sees your PLLAY secret key.
import { PLLAYBrowserClient } from "@pllay/sdk/browser";

const pllay = new PLLAYBrowserClient({ baseUrl: "/api/pllay" });
const state = await pllay.interactions.get(interactionId);
```

### Why not a public token?

A future PLLAY release may add a scoped, read-only, embeddable token type designed for direct browser use (similar in spirit to a Stripe publishable key). Until then, the proxy pattern above is the only safe design — and the SDK's browser entry point is built to make the unsafe alternative (passing a secret key to browser code) a compile error, not just a documentation warning.

## Webhook secrets

Your `webhook_secret` (returned once, when you set your webhook URL) is exactly the same class of credential as your API key — server-only. `webhooks.verify()`/`.constructEvent()` are exported **only** from `@pllay/sdk` (the Node.js entry), never from `@pllay/sdk/browser`. See [the webhook example](../examples/webhook-node) for a working handler.
