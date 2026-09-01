# Getting started

## Install

```bash
npm install @pllay/sdk
# or
pnpm add @pllay/sdk
# or
yarn add @pllay/sdk
```

Requires Node.js 18+ (for global `fetch`/`AbortController`/`crypto`). On older Node, pass a `fetch` polyfill via `PLLAYClient`'s `fetch` config option.

## Architecture

The package has two entry points:

```ts
import { PLLAYClient } from "@pllay/sdk";           // Node.js — holds your secret key
import { PLLAYBrowserClient } from "@pllay/sdk/browser"; // browser-safe — no secret key possible
```

Both are built from the same TypeScript source, published as ESM (`.mjs`) and CommonJS (`.cjs`) with full `.d.ts` types, and have **zero runtime dependencies**. Pick your entry point based on where the code runs, not based on which methods you need — both expose the same `interactions`/`outcomes`/`campaigns`/`analytics` module shape (the browser client omits `webhooks`, which is Node-only).

```
@pllay/sdk (index.ts)              @pllay/sdk/browser (browser.ts)
  ├─ PLLAYClient                     ├─ PLLAYBrowserClient
  ├─ interactions                    ├─ interactions
  ├─ outcomes                        ├─ outcomes
  ├─ campaigns                       ├─ campaigns
  ├─ analytics                       ├─ analytics
  ├─ webhooks  ← Node.js only        │  (not present — see docs/authentication.md)
  └─ all types + errors              └─ all types + errors
```

## Your first call

```ts
import { PLLAYClient } from "@pllay/sdk";

const pllay = new PLLAYClient({
  apiKey: process.env.PLLAY_API_KEY!,
  environment: "sandbox",
});

async function main() {
  const interaction = await pllay.interactions.create(
    {
      creatorExternalId: "creator-123",   // must already be linked to your partner account
      question: "Will the home team win?",
      options: ["Yes", "No"],
      timerSeconds: 60,
    },
    { idempotencyKey: crypto.randomUUID() },
  );

  console.log("Created:", interaction.interactionId);
  console.log("Public URL:", interaction.interactionUrl);
}

main().catch(console.error);
```

Run it:

```bash
PLLAY_API_KEY=pllay_sk_sandbox_your_key_here node your-script.js
```

If `creatorExternalId` isn't yet linked to your partner account, this throws `PLLAYNotFoundError` with code `creator_not_linked` — see [Sandbox guide](sandbox.md) for how to get a real sandbox partner + linked creator provisioned.

## Handling errors

Every failure is a typed subclass of `PLLAYError`:

```ts
import { PLLAYClient, PLLAYError, PLLAYRateLimitError, PLLAYValidationError } from "@pllay/sdk";

try {
  await pllay.interactions.create({ creatorExternalId: "", question: "", options: [] });
} catch (err) {
  if (err instanceof PLLAYValidationError) {
    console.error("Fix these fields:", err.fieldErrors);
  } else if (err instanceof PLLAYRateLimitError) {
    console.error(`Rate limited — retry after ${err.retryAfterSeconds}s`);
  } else if (err instanceof PLLAYError) {
    console.error(`PLLAY error [${err.code}]: ${err.message} (request ${err.requestId})`);
  } else {
    throw err; // not a PLLAY error at all
  }
}
```

## Next steps

- [Authentication](authentication.md) — understand key types, scopes, and safe browser use
- [Sandbox guide](sandbox.md) — get a real key and test data
- [API examples](api-examples.md) — every method, with realistic input/output
- [Webhooks](webhooks.md) — receive and verify event deliveries
- [Rate limits](rate-limits.md) — per-route limits and how retries interact with them
- [Going to production](production-launch.md) — when you're ready to move off sandbox
