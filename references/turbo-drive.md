# Turbo Drive

Drive intercepts link clicks and form submissions, fetches the response, and
replaces `<body>` while merging `<head>`. It is on by default once turbo is
loaded; most of the work here is *telling it what not to do*.

## Contents

- [Visits](#visits)
- [Cache and preview](#cache-and-preview)
- [Forms and redirects](#forms-and-redirects)
- [Attributes](#attributes)
- [Meta tags](#meta-tags)
- [Prefetch and preload](#prefetch-and-preload)
- [Page refreshes and morphing](#page-refreshes-and-morphing)
- [Events](#events)
- [Programmatic API](#programmatic-api)
- [What Drive ignores](#what-drive-ignores)

## Visits

Two kinds, and the distinction explains most surprising behavior:

- **Application visit** — a link click or `Turbo.visit()`. Always issues a network
  request. Default action is `advance` (pushes a history entry); `replace` swaps
  the current entry instead.
- **Restoration visit** — Back/Forward. Turbo restores from its snapshot cache
  when it has one, *then* may not re-request at all. This is why "my page is stale
  after going back" happens: you're seeing a cached snapshot.

## Cache and preview

Before leaving a page, Turbo snapshots it. On a later visit to that URL it may
display the snapshot immediately as a **preview** while the fresh response is in
flight — so a visitor can briefly see old content, and your JS can run against a
DOM that is about to be thrown away.

Handling it:

- `document.documentElement.hasAttribute("data-turbo-preview")` is true during a
  preview render — guard expensive or side-effecting setup with it.
- `data-turbo-temporary` on an element removes it from the snapshot, so
  flash messages don't reappear when the user navigates back.
- `<meta name="turbo-cache-control" content="no-cache">` opts a page out of being
  cached; `no-preview` allows caching but never shows it as a preview.
- `<meta name="turbo-visit-control" content="reload">` forces a full browser load
  when navigating to that page — the escape hatch for pages Turbo can't handle.

## Forms and redirects

The rules that cause the most confusion:

| Situation | Required response | Symptom if wrong |
|---|---|---|
| Successful POST/PATCH/DELETE | redirect (Turbo follows as 303) | nothing happens, page sits still |
| Failed submit (validation) | render with 4xx/5xx, conventionally `422 :unprocessable_entity` | errors never appear |
| Want a stream instead of navigation | `Content-Type: text/vnd.turbo-stream.html` | — |
| GET form rendering into a frame | `data-turbo-frame="…"` on the form | full page navigates |

Turbo requests non-GET form responses with the stream MIME type in the Accept
header automatically. GET requests do **not** accept streams unless the form or
link carries `data-turbo-stream`.

`data-turbo-submits-with="Saving…"` swaps a submit button's label while in flight —
cheaper than a Stimulus controller for the common case.

## Attributes

Applied to an element or any ancestor unless noted.

- `data-turbo="false"` — opt a link/form/subtree out of Drive entirely. Re-enable
  a descendant with `data-turbo="true"`.
- `data-turbo-action="replace" | "advance"` — history behavior for this visit.
- `data-turbo-method="delete"` — issue a non-GET request from a link. Prefer a real
  `button_to` form when you can: it works without JavaScript and is honest about
  being a mutation.
- `data-turbo-confirm="Are you sure?"` — native confirm dialog before the visit.
  Override globally with `Turbo.setConfirmMethod(fn)` to use your own modal.
- `data-turbo-track="reload"` — on `<script>`/`<link>` in `<head>`. When the asset
  URL changes between responses, Turbo does a full page load so users don't run
  stale assets against a new backend.
- `data-turbo-track="dynamic"` — head element is removed when absent from the next
  response.
- `data-turbo-permanent` — element (matched by id) is carried across renders
  untouched: flash containers, media players, open dropdowns.
- `data-turbo-temporary` — element is stripped from the cached snapshot.
- `data-turbo-preload` — fetch this link's page into the cache ahead of time.
- `data-turbo-prefetch="false"` — disable hover-prefetch for this link.
- `data-turbo-eval="false"` — don't re-execute this script on render.

## Meta tags

- `<meta name="turbo-root" content="/app">` — Drive only handles same-origin URLs
  under this path.
- `<meta name="turbo-prefetch" content="false">` — disable hover prefetching site-wide.
- `<meta name="turbo-refresh-method" content="morph">` — see morphing below.
- `<meta name="turbo-refresh-scroll" content="preserve">`
- `<meta name="turbo-cache-control" content="no-cache|no-preview">`
- `<meta name="turbo-visit-control" content="reload">`
- `<meta name="view-transition" content="same-origin">` — opt into the browser
  View Transition API for cross-document animation.

## Prefetch and preload

Prefetching on hover is **on by default** (~100ms hover delay). It makes
navigation feel instant, at the cost of extra requests. Turn it off for links
whose GET has side effects or is expensive — and note that a GET with side effects
is a bug independent of Turbo.

## Page refreshes and morphing

With `turbo-refresh-method` set to `morph`, a refresh of the *current* URL updates
only changed nodes (via idiomorph) instead of replacing `<body>`. Focus, scroll,
and open UI state survive.

```html
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
```

- `data-turbo-permanent` excludes a subtree from morphing — use it for third-party
  widgets, open popovers, anything with client state the server can't express.
- A `<turbo-stream action="refresh">` triggers this server-side; it accepts
  `method="morph|replace"` and `scroll="preserve|reset"`.
- turbo-rails' `broadcasts_refreshes` on a model broadcasts a debounced refresh to
  subscribers instead of a hand-written stream per attribute.
- Turbo suppresses the refresh triggered by the client's *own* request via a
  request id, so the actor doesn't double-render.

Interaction with Stimulus: morphing keeps existing elements, so a controller on an
untouched element is **not** disconnected and reconnected. Don't rely on
`connect()` re-running after a refresh. If an element is replaced, the controller
does reconnect — write `connect()`/`disconnect()` to be symmetric and idempotent
either way.

## Events

Dispatched on `document` unless the target is more specific. The useful ones:

| Event | When | Notes |
|---|---|---|
| `turbo:click` | link clicked | cancelable |
| `turbo:before-visit` | before an application visit | cancelable |
| `turbo:visit` | visit starts | |
| `turbo:before-cache` | before the page is snapshotted | tear down client state here |
| `turbo:before-render` | new body about to render | `event.detail.newBody`; can pause |
| `turbo:render` | after render | fires for preview too |
| `turbo:load` | after the visit completes | fires on initial load and every visit |
| `turbo:before-fetch-request` | outgoing request | `event.detail.fetchOptions.headers`; pausable via `resume()` |
| `turbo:before-fetch-response` | response received | |
| `turbo:submit-start` / `turbo:submit-end` | form lifecycle | `detail.formSubmission` |
| `turbo:frame-render` / `turbo:frame-load` | per frame | |
| `turbo:frame-missing` | response lacked the frame | cancelable — handle or Turbo errors |
| `turbo:before-stream-render` | stream about to apply | |
| `turbo:fetch-request-error` | network failure | |

`turbo:load` is not `DOMContentLoaded`. Code bound in `DOMContentLoaded` runs once
for the life of the tab; code bound on `turbo:load` runs on every visit, which is
how you end up with duplicated event listeners. The right answer for per-element
behavior is almost never either one — it's a Stimulus controller.

## Programmatic API

```js
import { Turbo } from "@hotwired/turbo-rails"

Turbo.visit("/path", { action: "replace" })
Turbo.cache.clear()
Turbo.session.drive = false          // disable Drive at runtime
Turbo.setConfirmMethod(async (message) => myModal(message))
Turbo.renderStreamMessage(html)      // apply a stream received out of band
```
