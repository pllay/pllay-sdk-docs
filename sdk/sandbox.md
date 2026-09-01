# Sandbox guide

## The honest state of things

PLLAY's hardened Partner API (`/v1/*`) is deployed to **both** environments: **sandbox** and **production**. Set `environment: "production"` once you have a `pllay_sk_live_...` key — see [`production-launch.md`](production-launch.md) for the partner go-live checklist.

**Practically: use `environment: "sandbox"` (the default) while you're building and testing, then switch to `"production"` for real traffic once you're ready — see [Going to production](production-launch.md).**

```ts
const pllay = new PLLAYClient({
  apiKey: process.env.PLLAY_API_KEY!,
  environment: "sandbox", // this is already the default — shown for clarity
});
```

## Getting a sandbox key

Sandbox keys (`pllay_sk_sandbox_...`) are issued by PLLAY via the `issue_partner_api_key` RPC, which is admin-only today — there is no self-service signup flow yet. Contact your PLLAY partner engineering contact with:

1. Your partner account name (or ask to have one created)
2. The scopes you need (see [authentication.md](authentication.md) for the list)
3. At least one creator `external_id` you want linked, so `interactions.create()` has someone to attribute an interaction to

Every real interaction (`interactions.create()`) requires `creatorExternalId` to already be linked to your partner account via an active `partner_creator_links` row — this is a fail-closed binding enforced by the platform itself, not something the SDK can route around. Without a link, `create()` throws `PLLAYNotFoundError` with code `creator_not_linked`. Ask for at least one linked creator when you request your sandbox key.

## Verifying your setup

```ts
import { PLLAYClient } from "@pllay/sdk";

const pllay = new PLLAYClient({ apiKey: process.env.PLLAY_API_KEY! });

const config = await pllay.campaigns.get();
console.log("Connected as partner:", config.partnerId);
```

If this throws `PLLAYAuthError`, double check the key was copied exactly (including its `pllay_sk_sandbox_` prefix) and hasn't been revoked.

## Running this SDK's own sandbox test suite

The SDK's automated tests are split in two:

- **Offline tests** (`pnpm test`) — run against a local, in-process contract server (`test/contract-server.ts`) that mirrors the real API's route shapes. Fast, deterministic, no credentials needed, run on every CI build.
- **Sandbox integration tests** (`pnpm test:sandbox`) — real network calls against the actual PLLAY sandbox. Skipped by default; requires real credentials via environment variables (never hardcoded — see below):

```bash
PLLAY_SDK_RUN_SANDBOX_TESTS=1 \
PLLAY_SANDBOX_API_KEY=pllay_sk_sandbox_your_real_key \
PLLAY_SANDBOX_CREATOR_EXTERNAL_ID=your_linked_creator_external_id \
pnpm test:sandbox
```

This is the suite that was actually run — with real, provisioned, since-cleaned-up sandbox credentials — before this SDK's 0.1.0 release, satisfying "do not publish until all examples execute successfully against the approved sandbox." See `CHANGELOG.md` and the release checklist for the record of that run.

## What sandbox data looks like

Sandbox interactions settle exactly like production ones — they use the same platform-wide settlement path, the same trigger, the same revenue-split math — they're just attributed to a `sandbox`-environment key rather than `live`. There is no separate sandbox database; it's the same platform, environment-scoped by key. Don't create sandbox interactions you don't intend to clean up if you're running your own exploratory scripts — see the SDK's own test fixtures (`test/sandbox.integration.test.ts`) for the pattern of provisioning-then-cleaning-up used in this SDK's own release process.
