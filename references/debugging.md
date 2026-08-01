# Debugging Hotwire

Much of Hotwire fails quietly: a stream targeting an id that isn't in the DOM, a
controller whose filename doesn't match its identifier, an action bound to a
method that doesn't exist. Nothing throws; the page just sits there. So debug by
observation — look at the actual network response and the actual DOM before
forming a theory.

But **not everything is silent, and the difference is diagnostic.** Frame-missing
in Turbo 8 paints `Content missing` into the frame and throws
`TurboFrameMissingError`. So "nothing happened and the console is clean" actively
*rules out* the most-cited frame explanation rather than confirming it. Check what
your bundled Turbo actually does before leaning on any of this — the version in
`node_modules/@hotwired/turbo` or the built bundle is the ground truth, and it
takes one grep:

```bash
grep -n "Content missing\|TurboFrameMissingError" app/assets/builds/*.js node_modules/@hotwired/turbo/dist/*.js
```

Reasoning from what a framework "generally does" is how you end up confidently
diagnosing the wrong thing. The source is right there.

**This file is for you to work through, not to recite.** Walk the relevant tree,
then reply with the cause and the fix — a sentence and a diff. Pasting the ranked
hypotheses, the branch you ruled out, and the full check procedure back to the
person just makes them do the filtering you were supposed to do.

**Work the tree yourself before you answer.** Most branches below resolve from the
repo in seconds — does that template render the frame id, does the controller
short-circuit, is the identifier spelled the way the filename implies. Read those
files. The point of skipping the recitation is that you already did the checking,
not that nobody does it. Report the cause with the evidence that settled it, and
if a branch needs something you can't reach — their network tab, their logs, a
browser — say so and name the one check that decides it, rather than picking the
likeliest story and asserting it.

## First moves, always

1. **Network tab → the request → Response.** Is it HTML? What status? Does it
   contain the frame id or `<turbo-stream>` you expect? This answers most
   questions in ten seconds and it is evidence, not inference.
2. **Console.** Turbo logs frame-missing errors and Stimulus logs registration
   problems there.
3. **Elements tab.** Does the target id exist in the DOM *at the moment the
   response arrives*? Ids inside a collapsed frame or a not-yet-loaded lazy frame
   don't exist yet.

Turn on Turbo's own logging while investigating:

```js
Turbo.session.drive // confirm Drive is enabled
document.addEventListener("turbo:before-fetch-request", e => console.log("→", e.detail.url.href))
document.addEventListener("turbo:before-stream-render", e => console.log("stream", e.target.action, e.target.target))
document.addEventListener("turbo:frame-missing", e => console.warn("frame missing", e.detail.response.url))
```

---

## Symptom: form submits, nothing happens

Check in this order:

1. **Status code.** Success must redirect; Turbo will not navigate on a 200 body
   after POST. Failure must be 4xx (`status: :unprocessable_entity`) or the
   re-rendered form with errors is discarded.
2. **Redirect target renders the frame** (if the form is in a frame). A redirect to
   a page without that frame id is silently dropped.
3. **`format.turbo_stream` block exists** if you're returning a stream, and the
   `respond_to` isn't falling through to `format.html`.
4. Server log: did the action even run, or did a `before_action` redirect it?

## Symptom: link inside a frame reloads the whole page

- The link has `data-turbo-frame="_top"`, or the frame itself has `target="_top"`.
- The link goes cross-origin, or its path's last segment contains a `.` — Drive
  ignores those.
- `data-turbo="false"` on the link or an ancestor.

## Symptom: frame shows "Content missing" / `turbo:frame-missing` in console

This is the *loud* failure — the frame renders `Content missing` and Turbo throws.
If instead the frame is completely inert with a clean console, skip to the next
section; you are not in this branch.

- The response doesn't contain `<turbo-frame id="…">` with the *exact* same id.
  Compare them character by character — `dom_id` mismatches (`comment_42` vs
  `comments_42`) are the usual culprit.
- The request got redirected to sign-in and that page has no such frame. Fix
  server-side with `<meta name="turbo-visit-control" content="reload">` on the login
  page, or client-side by handling `turbo:frame-missing` and doing a full visit.
- The response was a 500 error page.

## Symptom: frame is completely inert — no request, no error, no visible change

Nothing in the network tab is the key observation: the click never became a fetch.
Turbo's frame guard requires the frame to be enabled, active, not already
`complete`, and to have a source URL, so check in that order:

- `data-turbo="false"` on the link, the frame, or any ancestor.
- The page is showing a **cached preview**: `data-turbo-preview` on `<html>` makes
  frames inert, with no request and nothing logged. Confirm in the console with
  `document.documentElement.dataset.turboPreview`. A visit whose fresh response
  never arrives can leave the page stuck in this state until a full reload — which
  matches "it heals after a hard refresh".
- The frame already carries `[complete]` and has no new `src`.
- JavaScript died earlier in the page lifecycle, so Turbo never wired anything up.
  One earlier uncaught exception disables everything downstream of it.
- The Edit control is a plain `<a>` outside any frame, so nothing was ever scoped
  to the frame in the first place.

## Symptom: stream response arrives but nothing changes

Confirm the response body in the Network tab first — if `<turbo-stream>` tags are
there, the server side is fine and the problem is the target.

- **Target id doesn't exist in the DOM.** A stream targeting a missing id is a
  silent no-op. Check for typos and for elements inside a not-yet-loaded lazy frame.
- **Wrong MIME type.** The response must be `text/vnd.turbo-stream.html`. If you
  hand-built the response or used `render plain:`, it will be ignored.
- **GET request.** Turbo doesn't accept streams for GET unless the form/link has
  `data-turbo-stream`.
- **`<template>` missing.** Hand-written stream markup needs content wrapped in
  `<template>` for every action except `remove` and `refresh`.

## Symptom: broadcast never arrives

- Is `<%= turbo_stream_from … %>` in the rendered page, inside `<body>`, and not
  inside a `data-turbo-permanent` region?
- Same stream name on both sides? `turbo_stream_from @post` and
  `broadcast_append_to @post` must resolve to the same signed stream name.
- Broadcasting from `after_create_commit`, not `after_save`?
- Development Action Cable adapter: the default `async` adapter is per-process, so
  a broadcast from a background job process never reaches the web process. Switch
  to `redis` to test this properly.
- With `_later_to`, is the job queue actually running?
- Does the broadcast partial reference `current_user` or other request state? It
  renders with no request; that raises inside the job and the broadcast dies
  silently from the browser's point of view. Check the job logs.

## Symptom: Stimulus controller never connects

- **Filename ↔ identifier.** `date_picker_controller.js` → `data-controller="date-picker"`.
  Wrong pairing is the single most common cause.
- Is the file in the directory the app's controller loader scans, and did the
  asset build actually rebuild?
- `console.log(this.identifier)` in `connect()` — if it never prints, it's
  registration; if it prints, the problem is downstream.
- `Stimulus.debug = true` in the console logs every connect/disconnect/action.
- Typo in the target attribute — `data-target="…"` (old Stimulus 1 syntax) instead
  of `data-[identifier]-target="…"`.

## Symptom: handler fires 2, 3, N times (growing with navigation)

Almost always one of:

- Event listeners bound on `turbo:load` or `DOMContentLoaded` and never removed —
  each visit adds another. Move the behavior into a Stimulus controller.
- `addEventListener` in `connect()` with no `removeEventListener` in `disconnect()`.
  Use `data-action` with `@window`/`@document`, which unbinds automatically.
- A controller mounted on an element that a stream `replace`s repeatedly, plus
  window-level listeners.

## Symptom: client state lost on every update

- Turbo Stream `replace` destroys the element (and its Stimulus controller). Use
  `update` to keep the element and swap only its contents.
- Full page renders discard everything not marked `data-turbo-permanent`.
- Morphing preserves untouched nodes but will still rewrite attributes it thinks
  changed; mark widget containers `data-turbo-permanent`.

## Symptom: content is stale after pressing Back

That's a restoration visit rendering a cached snapshot — working as designed. If
the page must always be fresh, add `<meta name="turbo-cache-control" content="no-cache">`.
If it's just a flash message reappearing, mark it `data-turbo-temporary`.

## Symptom: it works in dev, breaks in production

- `data-turbo-track="reload"` on assets is doing its job: after a deploy the asset
  digests change and Turbo forces a full reload. Expected.
- Action Cable adapter differs between environments (`async` vs `redis`).
- Background jobs run inline in dev (`_later_to` broadcasts work) but through a
  real queue in production (they don't, if the worker is down).
- CDN or proxy stripping/ignoring the `Vary: Accept` header, so a cached HTML
  response is served where a turbo_stream was requested.

## When you still can't see it

Reproduce it in a request spec. Assert on the actual response body — the status,
the MIME type, the presence of the frame id or stream tag. If the spec passes, the
bug is in the browser layer (ids, timing, Stimulus); if it fails, it's server-side
and you've just cut the search space in half.
