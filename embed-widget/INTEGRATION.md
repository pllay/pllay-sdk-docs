# Integration guide

## 1. Load the widget

```html
<script src="https://cdn.pllay.io/embed-widget/loader.js" defer></script>
```

The loader lazily imports the real widget bundle only once a
`<pllay-interaction>` element scrolls near the viewport (`rootMargin:
200px`). It also picks up elements added later by your own JS via a
`MutationObserver`, so it works with client-side routing.

If you're bundling the widget yourself instead of using the CDN loader:

```js
import "@pllay/embed-widget"; // registers the custom element immediately, no lazy-load
```

## 2. Get an interaction id and embed token

`POST /v1/predictions/launch` (authenticated with your partner secret
key, **server-side only**) returns both in its response:

```json
{
  "prediction_id": "8aa8...",
  "embed_token": "pllay_et_...",
  "embed_token_expires_at": "2026-09-28T04:29:56Z"
}
```

Pass `prediction_id` and `embed_token` straight through to your
rendered page. **Never** put your partner secret key (`pllay_sk_...`)
anywhere the browser can read it — the embed token is the only
credential this component ever accepts, and it is scoped read-only to
this one interaction.

## 3. Render it

```html
<pllay-interaction
  interaction-id="381d79d5-fbb8-459a-ab75-4359496eb1b4"
  embed-token="pllay_et_livetestfixturedonotrevoke0000000000000001"
></pllay-interaction>
```

### Attributes

| Attribute | Required | Values | Default |
|---|---|---|---|
| `interaction-id` | yes | UUID | — |
| `embed-token` | yes | `pllay_et_...` | — |
| `locale` | no | BCP-47 tag, e.g. `en`, `es`, `fr`, `ar` | `en` |
| `theme` | no | `light` \| `dark` \| `auto` | `auto` (`prefers-color-scheme`) |
| `display-mode` | no | `card` \| `compact` \| `fullwidth` | `card` |
| `api-base-url` | no | URL | production `partner-api` |
| `participate-base-url` | no | URL prefix; interaction id appended | `https://pllay.io/p` |
| `consent` | no | `"true"` \| `"false"` | `"false"` |

No attribute or JS property accepts a partner secret key — that is a
structural property of the type, not a documented restriction to
remember to follow.

### Events

Every event is a `CustomEvent`, `bubbles: true, composed: true` (crosses
the shadow boundary), carrying no PII (a public embed has no fan
identity):

| Event | Fires when | `detail` |
|---|---|---|
| `ready` | first successful state fetch | `{ interactionId }` |
| `viewed` | the element is ≥50% in the viewport (once) | `{ interactionId }` |
| `joined` | the "Participate on PLLAY" CTA is clicked | `{ interactionId }` |
| `locked` | `widget_state` transitions to `locked` | `{ interactionId }` |
| `resolved` | `widget_state` transitions to `verified_result` | `{ interactionId, winner, winningOptionIndex }` |
| `error` | fetch/auth/config failure | `{ interactionId, code, message, retryable }` |
| `submitted`, `rewarded` | reserved, not emitted in this version | — see the README's Known limitations |

```js
document.addEventListener("resolved", (e) => {
  console.log(e.detail.winner);
});
```

## 4. Theme it

Every visual token is a CSS custom property set on the host element —
Shadow DOM means nothing else in your page's CSS can reach inside the
widget, and the widget's own disclosure line has no `part` attribute, so
it cannot be styled or hidden from outside either.

```css
pllay-interaction {
  --pllay-color-accent: #7c5cff;
  --pllay-color-bg: #0e0e12;
  --pllay-font-family: "Inter", sans-serif;
  --pllay-radius: 8px;
  --pllay-logo-url: url("/your-logo.svg");
}
```

Full token list: `--pllay-color-{bg,surface,text,text-muted,accent,
border,success,warning,danger}`, `--pllay-font-family`,
`--pllay-font-family-mono`, `--pllay-radius`, `--pllay-space-{1,2,3,4}`,
`--pllay-logo-url`.

Sponsor attribution (shown in the disclosure line, not removable) is set
as a JS property, since it's plain text rather than a stylesheet
concern:

```js
document.querySelector("pllay-interaction").sponsorLabel = "Acme Corp";
```

## 5. Locale and RTL

```html
<pllay-interaction locale="es" ...></pllay-interaction>
```

Built-in: `en`, `es`, `fr`. Any other locale falls back to English (with
a console warning) unless you supply overrides:

```js
document.querySelector("pllay-interaction").strings = {
  open: "متاح للمشاركة",
};
```

`ar`, `he`, `fa`, `ur` set `dir="rtl"` on the host automatically,
regardless of whether translated strings are supplied.

## 6. Analytics (opt-in, off by default)

```html
<pllay-interaction consent="true" ...></pllay-interaction>
<script>
  document.querySelector("pllay-interaction").analyticsEndpoint =
    "https://your-analytics.example/beacon";
</script>
```

With no `analyticsEndpoint` set, or `consent` left at its default
`"false"`, the widget makes zero analytics network calls — every DOM
event above still fires regardless, since that's how your own page
observes the widget; this toggle only controls a *separate*
`navigator.sendBeacon()` call to whatever endpoint you configure.

## 7. CSP

```
Content-Security-Policy:
  script-src 'self' https://cdn.pllay.io;
  connect-src 'self' https://api.pllay.io;
```

No inline scripts, no `eval`, no `Function()` anywhere in this package.
Styles are applied via Constructable Stylesheets (falls back to a
`<style>` tag pre-Safari 16.4), which don't require `unsafe-inline` in
`style-src`.

## 8. React

```tsx
import { PLLAYInteraction } from "@pllay/embed-widget/react";

<PLLAYInteraction
  interactionId="381d79d5-fbb8-459a-ab75-4359496eb1b4"
  embedToken="pllay_et_livetestfixturedonotrevoke0000000000000001"
  onResolved={(d) => console.log(d.winner)}
/>;
```

The wrapper is a thin adapter over the custom element — it sets no
behavior of its own.
