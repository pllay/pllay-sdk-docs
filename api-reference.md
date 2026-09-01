# PLLAY Partner API

The white-label B2B Partner API (`supabase/functions/partner-api`), hardened
2026-08-29 for paid enterprise Proof Sprints. This document describes only
routes that are **implemented and deployed** — see
[`docs/openapi/partner-api-v1.yaml`](openapi/partner-api-v1.yaml) for the
machine-readable spec of the `/v1/*` surface, contract-tested against the
implementation in `supabase/functions/partner-api/openapi-contract.test.ts`.

This is a different, older, `predictions`-based surface from the
org/moment-based resource model documented in
[`docs/ENTERPRISE_API.md`](ENTERPRISE_API.md) / `docs/openapi/pllay-v1.yaml`.
That design is unimplemented. This one is real, deployed, and now hardened.

## Versioning

- **`/v1/*`** — the hardened, scoped, current surface. Use this for anything new.
- **Unversioned** (`/predictions/launch`, `/predictions/{id}/state`,
  `/predictions/{id}/settle`, `/creators/{external_id}/stats`,
  `/revenue/summary`) — the original routes, preserved byte-for-byte for
  backward compatibility, considered **deprecated**. New integrations should
  use `/v1/*`. No sunset date has been set; when one is, it will be
  communicated to every partner with an active legacy-format (`pllay_pk_`)
  key via their registered contact and a `Sunset` response header added
  ahead of the change.
- **Breaking-change policy**: a breaking change to `/v1/*` becomes `/v2/*`.
  Additive changes (a new optional field, a new endpoint, a new scope) ship
  in place under `/v1/*`.

## Authentication

Header: `X-Partner-Key: <key>`

Two key formats are both accepted, on every route (legacy and `/v1/*` alike):

| Prefix | Scope model | Environment | Where issued |
|---|---|---|---|
| `pllay_pk_...` | none — full access (predates scopes) | production only | `admin_create_b2b_partner` (unchanged) |
| `pllay_sk_live_...` | explicit, enforced | production | `issue_partner_api_key(..., 'production', ...)` |
| `pllay_sk_sandbox_...` | explicit, enforced | sandbox | `issue_partner_api_key(..., 'sandbox', ...)` |

A legacy `pllay_pk_` key is treated as carrying every scope, so it works on
every `/v1/*` route unchanged — hardening never downgrades what an existing
key can already do.

Keys are hashed at rest (`sha256`, `extensions.digest`) — never stored or
recoverable in plaintext after issuance. `issue_partner_api_key` returns the
plaintext exactly once, at creation.

### Key lifecycle (admin-only RPCs, not yet exposed via edge function — call directly)

```sql
select * from issue_partner_api_key(
  _partner_id => '<uuid>',
  _environment => 'sandbox',            -- or 'production'
  _scopes => array['interactions:read', 'interactions:write', 'outcomes:write'],
  _expires_at => null,                  -- or a timestamptz
  _caller => auth.uid()
);
-- => { key_id, api_key, key_prefix }   -- api_key shown ONCE, here

select * from rotate_partner_api_key(_key_id => '<uuid>', _caller => auth.uid());
-- issues a replacement and revokes the old key atomically -- no overlap window

select revoke_partner_api_key(_key_id => '<uuid>', _reason => 'partner offboarded', _caller => auth.uid());
```

`last_used_at` / `last_used_ip` are updated on every successful `/v1/*` auth
call, so a stale or compromised key is identifiable from `partner_api_keys`
without instrumenting the caller.

### Scopes

| Scope | Grants |
|---|---|
| `interactions:read` | `GET /v1/predictions`, `.../state` |
| `interactions:write` | `POST /v1/predictions/launch`, `PATCH .../close` |
| `outcomes:read` | `GET /v1/predictions/{id}/resolution` |
| `outcomes:write` | `POST /v1/predictions/{id}/outcome` |
| `analytics:read` | `GET /v1/analytics` |
| `campaigns:read` | `GET /v1/campaign-config` |
| `campaigns:write` | reserved — no route currently requires it |
| `webhooks:manage` | `GET /v1/webhooks`, `PATCH /v1/webhooks` |
| `intelligence:read` | `GET /v1/intelligence/summary`, `GET /v1/intelligence/moments`, `GET /v1/intelligence/audience` |
| `intelligence:benchmarks` | reserved — no route currently requires it (cross-title benchmarking, see docs/PLATFORM_INTELLIGENCE_SPEC.md §4.3) |

Renamed 2026-08-31 to match the platform's canonical scope vocabulary
(`interactions:*` / `outcomes:*` rather than `predictions:*`); `campaigns:write`
was dropped entirely (no route ever required it). Safe to rename in place —
zero live partners had ever issued a scoped `pllay_sk_` key at the time of the
rename (see `supabase/migrations/20260831025652_rename_partner_api_scopes.sql`).

A key lacking a route's required scope gets the **same** `401 invalid_api_key`
response as a nonexistent or revoked key — scope violations are not
distinguishable from auth failures, so a stolen key's exact permission
boundary can't be probed by an attacker.

## Errors

Every `/v1/*` response — success or failure — carries `X-Request-Id`. Errors:

```json
{
  "error": {
    "code": "not_found",
    "message": "prediction not found",
    "request_id": "2fa7...",
    "documentation_url": "https://pllay.io/developers/partner-api#not_found",
    "retryable": false,
    "field_errors": [{ "field": "options", "message": "must be an array of 2-4 non-empty strings" }]
  }
}
```

`field_errors` is present only on `validation_failed`. `retryable` is `true`
for `rate_limited` / `internal_error`, `false` for everything else — a
caller can safely automate retry-with-backoff on `retryable: true` and
should surface everything else to a human.

Legacy (unversioned) routes are **unchanged**: `{ error: "some_code" }`, no
`request_id`/`retryable`/envelope. Do not build a client against both route
families expecting one error shape.

## Anti-enumeration

On `/v1/*`, a prediction that **exists but belongs to a different partner**
returns the identical `404 not_found` as a prediction that doesn't exist at
all. This is deliberate: distinguishing "not yours" (403) from "doesn't
exist" (404) lets a caller enumerate valid prediction IDs belonging to other
partners by status code alone. The legacy unversioned
`/predictions/{id}/state` and `/predictions/{id}/settle` routes still return
`403 forbidden` for "not yours" — that is unchanged, for backward
compatibility with existing integrations that may already branch on it.

## Idempotency

Header: `Idempotency-Key: <any string you choose>` — supported on every
mutating `/v1/*` route (`launch`, `close`, `outcome`, `webhooks`).

- Same key + same request body, replayed within 24h → the original response
  is replayed verbatim (no new prediction created, no double-settlement).
- Same key + a **different** request body → `409 idempotency_key_conflict`.
- No key supplied → every request is processed independently, as before.

Scoped per-partner, so two different partners can safely reuse the same
literal key string without colliding.

## Pagination

`GET /v1/predictions?limit=25&cursor=<opaque>` — cursor-based, stable under
concurrent inserts (sorted on `created_at, id`). Response:

```json
{ "data": [...], "pagination": { "next_cursor": "eyJ...", "has_more": true } }
```

Pass `next_cursor` back as `cursor` for the next page. `next_cursor` is
`null` when `has_more` is `false`.

## Rate limits

Per-partner, per-route, fixed 1-minute buckets (`check_rate_limit`). Exceeding
returns `429 rate_limited` (retryable) with `Retry-After` seconds.

| Route class | Limit |
|---|---|
| Any `GET` on `/v1/*` | 120/min |
| `PATCH /v1/webhooks` | 10/min |
| Every other `/v1/*` write | 30/min |
| Any legacy (unversioned) route | 60/min (unchanged) |

Rate limiting **fails open** — an RPC error never blocks a legitimate
request; it's defense in depth, not the primary access control.

## Request size limit

Any request body over 100,000 bytes is rejected with `413 payload_too_large`
before JSON parsing — applies to every route.

## What this API does NOT expose

Per the hardening brief's explicit exclusion list, no `/v1/*` route ever
returns or accepts:

- Internal settlement mechanics beyond `service_fee_cents` /
  `partner_revenue_cents` / `pllay_revenue_cents` (already-public per-partner
  splits — no ledger internals, no other partners' figures)
- Fraud signals, fraud-review state, or anything from `fraud_*` tables
- Fan/user PII (name, email, phone, address) — predictions surface only
  aggregate `total_pool` / `total_participants`, never individual bettors
- Reward or webhook secrets, except the one-time plaintext returns
  (`issue_partner_api_key`'s `api_key`, `PATCH /v1/webhooks`'s
  `webhook_secret`) — never re-returned by any other call
- Any admin capability (partner creation, other partners' data, platform
  configuration)
- The raw contents of an intelligence export's storage object — only ever
  a signed, 15-minute URL minted fresh by `GET /v1/intelligence/exports/{id}`,
  never a stored/reusable link or a direct bucket path

## Endpoints

See [`docs/openapi/partner-api-v1.yaml`](openapi/partner-api-v1.yaml) for the
full request/response schema of every `/v1/*` route. Summary:

| Method | Route | Scope | Idempotent |
|---|---|---|---|
| POST | `/v1/predictions/launch` | `interactions:write` | yes |
| GET | `/v1/predictions` | `interactions:read` | — |
| GET | `/v1/predictions/{id}/state` | `interactions:read` | — |
| PATCH | `/v1/predictions/{id}/close` | `interactions:write` | yes |
| POST | `/v1/predictions/{id}/outcome` | `outcomes:write` | yes |
| GET | `/v1/predictions/{id}/resolution` | `outcomes:read` | — |
| GET | `/v1/campaign-config` | `campaigns:read` | — |
| GET | `/v1/analytics` | `analytics:read` | — |
| GET | `/v1/webhooks` | `webhooks:manage` | — |
| PATCH | `/v1/webhooks` | `webhooks:manage` | yes |
| GET | `/v1/intelligence/summary` | `intelligence:read` | — |
| GET | `/v1/intelligence/moments` | `intelligence:read` | — |
| GET | `/v1/intelligence/audience` | `intelligence:read` | — |
| POST | `/v1/intelligence/exports` | `intelligence:read` | yes |
| GET | `/v1/intelligence/exports/{id}` | `intelligence:read` | — |
| GET | `/v1/embed/state` | none — public embed token | — |

### Outcome submission has no propose/confirm split

`POST /v1/predictions/{id}/outcome` settles **immediately and
authoritatively** the moment it succeeds — there is no separate
propose-then-confirm step, unlike PLLAY Pulse's sponsor-outcome workflow.
Submit only once you are certain of the winning outcome.

### Platform Intelligence is a separate partner type, not another prediction scope

`GET /v1/intelligence/summary` is not part of the predictions-hosting
surface above — it's Phase 1 of the partner-facing product specced in
[`docs/PLATFORM_INTELLIGENCE_SPEC.md`](PLATFORM_INTELLIGENCE_SPEC.md). An
intelligence partner is a `b2b_partners` row with `product_domain =
'intelligence'` and `tenant_id` set (scoping which content sources it may
read); `take_rate`/`partner_take` are simply unused for this
`product_domain`, same as every other field a given partner type doesn't
need. A predictions-hosting partner's key is refused
(`intelligence_not_enabled`, 403) even if `intelligence:read` were
mistakenly granted to it — `product_domain` is a second, independent
gate, not just the scope check. The whole surface is additionally gated
by the `enterprise_api_enabled` capability flag. `summary`, `moments`,
and `audience` now exist; `benchmarks` and the export job are specced but
not yet built. `moments` deliberately never returns a raw creator
identity (`playable_events.creator_id`, an internal `auth.users` id) —
event-level ranking by `return_rate` proves tenant ownership on its own,
without resolving or exposing who the creator is.

All three routes apply the same k-anonymity floor as `brand_audience_insights`
(`public.graph_k_floor()`, default 25, hard minimum 5): `summary` suppresses
its whole payload's audience-behavior fields (`participants`,
`repeat_participants`, `interactions_per_participant`,
`session_retention_pct`) to `null` with `suppressed: true` when the window's
total participant count is below the floor; `moments` applies the same rule
per event (`event_id`/`event_type` stay visible, the rest goes `null` with
that moment's own `suppressed: true`); `audience` reuses
`brand_audience_insights`' own built-in suppression per cohort
(`campaign_id`/`content_category` stay visible, `cohort_size`/
`mean_accuracy`/`mean_events_per_fan` go `null` with that cohort's own
`suppressed: true`) rather than reimplementing it. All three responses
always carry `k_floor` so a caller knows the threshold being applied. This
is not missing data — it's the same "narrow-query cohort isolation"
protection every other brand/partner-visible aggregate in this repo
already enforces.

## Webhooks

Unchanged delivery mechanics: HMAC-SHA256 signed
(`X-Pllay-Signature: sha256=<hex>`), `X-Pllay-Event`, `X-Pllay-Delivery`
(unique per attempt), 3 attempts with backoff (0s / 4s / 16s), 8s timeout per
attempt. Events: `prediction.started`, `prediction.settled`.
`PATCH /v1/webhooks` lets a partner self-serve its `webhook_url` — every
change rotates the signing secret, returned once in the response.
`GET /v1/webhooks` reads the current configuration back — `webhook_url` and
a `configured` boolean, never `webhook_secret` itself (write-once/return-once:
re-exposing it on every read would defeat the point of rotating it).

**A separate, newer webhook system exists for a different, larger event
taxonomy** (`interaction.*`/`reward.*`/`campaign.*`) — see
[`PARTNER_WEBHOOK_SERVICE.md`](PARTNER_WEBHOOK_SERVICE.md). It does not
share config, delivery logs, or retry state with the `prediction.*` system
documented above; the two are deliberately independent, not a
migration-in-progress.

## Known limitations

- `issue_partner_api_key` / `rotate_partner_api_key` / `revoke_partner_api_key`
  are Postgres RPCs, not (yet) edge function routes — a partner cannot
  self-serve key issuance today; an operator runs the SQL above on their
  behalf. Exposing an authenticated admin-only HTTP wrapper is a reasonable
  next step once a real partner needs it.
- `intelligence:benchmarks` is defined but no route currently requires it —
  reserved for cross-title benchmarking once enough intelligence partners
  exist in one content category to form a peer set (spec §4.3/§6 Phase 2).
- No intelligence partner currently exists in production (zero
  `b2b_partners` rows with `product_domain = 'intelligence'`) — this route
  is built and self-tested, not yet exercised by a real partner.
- No full load test has been run (see the hardening report for what was
  actually measured — a small number of live smoke requests against
  staging, explicitly not a load test).
- `/v1/predictions` lists only `id, title, state, options,
  winning_option_index, total_pool, created_at, updated_at` — add fields to
  `partner_api_list_predictions` (migration 20260829025052) if a partner
  needs more.
