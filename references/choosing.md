# Where each tool's responsibility ends

The ladder in SKILL.md says take the lowest rung that works. This file is for when
two rungs both look plausible — the boundaries, each with the question that
actually settles it.

The framework's own framing, worth holding onto: Drive handles navigation, Frames
"compartmentalize the content and navigation for a fragment of the document",
Streams "change any part of the page in response to updates", and Stimulus works
"below the grade of a full page change". Three of those four are about *what
triggered the change*, not about what the change looks like. That's usually the
discriminator people miss.

## Contents

- [Drive ↔ Frame](#drive--frame)
- [Frame ↔ Stream](#frame--stream)
- [Stream ↔ morphing refresh](#stream--morphing-refresh)
- [Turbo ↔ Stimulus](#turbo--stimulus)
- [Stimulus ↔ the platform](#stimulus--the-platform)
- [Turbo Streams ↔ raw Action Cable](#turbo-streams--raw-action-cable)
- [Hotwire ↔ a client framework](#hotwire--a-client-framework)
- [Quick table](#quick-table)

## Drive ↔ Frame

**Question: after this interaction, is the user somewhere else?**

If the whole page now means something different — a different record, a different
section — that's a navigation, and Drive already handles it with a plain link and
a normal render. Wrapping most of the page in a frame to avoid a "reload" is
reinventing Drive while giving up a real URL, working Back, and the restoration
cache.

A frame earns its keep when the fragment navigates *and the surrounding document
must not*: an open filter panel, scroll position halfway down a long list, focus
in a search box, a video still playing. That's the documented purpose — frames
preserve "the rest of the document's state (for example, its current scroll
position or focused element)".

Adjacent pair:
- "Clicking a plot opens the plot page" → Drive. Nothing to preserve.
- "Clicking a plot in the sidebar shows it in the right pane while the sidebar
  keeps its scroll and selection" → Frame.

If a frame navigation *should* be linkable (tabs, pagination, filters), add
`data-turbo-action="advance"` rather than escalating to something heavier. That
is exactly the case that looks like it needs Drive and doesn't.

## Frame ↔ Stream

**Question: did a navigation happen inside one region, or did state change in
several places at once?**

A frame responds to *its own* navigation: a link or form inside it, or something
aimed at it with `data-turbo-frame`. One request, one region, and the response is
an ordinary page that happens to contain that frame.

A stream is a list of DOM operations addressed by id. Reach for it when one action
must touch regions that aren't nested together (the row *and* the counter *and*
the flash), or when nothing navigated at all because the change originated on the
server.

The rule that catches most confusion, straight from the handbook: **a frame that
exists only to be a stream target should not be a frame.** "If your application
utilizes `<turbo-frame>` elements for the sake of a `<turbo-stream>` element,
change the `<turbo-frame>` into another built-in element." A `<div id="comments">`
is a perfectly good stream target. Making it a frame adds navigation semantics you
then have to reason about — links inside it start behaving differently, and
`turbo:frame-missing` becomes a failure mode you didn't need.

Adjacent pair:
- "Approving a comment re-renders that comment row" → Frame.
- "Approving a comment re-renders the row *and* decrements the pending badge in
  the header" → Stream. The badge isn't inside the row's frame and never will be.

## Stream ↔ morphing refresh

Both keep the page in place; they differ in who decides what changed.

**Targeted streams** when you know exactly which elements move and the update is
frequent or small — a counter ticking, a row appending. Cheap per update, but you
maintain the target list by hand, and a missed target is a silent no-op.

**`broadcasts_refreshes` + morphing** when a change touches many parts of the page,
when what's visible differs per viewer (each client re-requests and renders its
own page, so per-user markup resolves itself), or when hand-maintaining stream
targets has become the bulk of the work. Costs a full render per client per
change, so it's the wrong default for a high-frequency ticker.

The per-viewer point is the one that decides real arguments: a broadcast stream
renders **once, without a request** — no `current_user`, no session. If the markup
differs per viewer you cannot broadcast one payload to everyone. Refresh, or
broadcast to a per-user stream.

## Turbo ↔ Stimulus

**Question: does the server have anything to say about this?**

If new or changed markup is involved, the server should produce it — Turbo
delivers it. Stimulus is for behavior that needs no round-trip: toggling a class,
focusing a field, debouncing input, reading a value out of the DOM, driving a
third-party widget.

Stimulus deliberately does not render. It focuses on "manipulating, not creating
elements" and explicitly rejects "turning JSON into DOM elements via a template
language". A controller that fetches JSON and builds HTML has taken over Turbo's
job, and it's the single most common way a Hotwire app goes wrong.

The two legitimately compose, and this is the shape to aim for: the server sends
markup that carries `data-controller`, and the controller adds behavior to it.
Turbo Streams cannot invoke custom JavaScript — "this is a feature, not a bug" —
so attaching behavior via attributes on the delivered HTML *is* the supported
mechanism, not a workaround.

Adjacent pair:
- "Picking a skill greys out the ones it conflicts with, instantly, no save" →
  Stimulus. No server opinion needed; the rules are already on the page.
- "Picking a skill recalculates remaining points and validates against the
  rulebook" → server round-trip. The rules live server-side and must agree with
  what gets saved.

## Stimulus ↔ the platform

**Question: does the browser already do this?**

Before writing a controller: `<details>`/`<summary>` for accordions and
disclosure, `<dialog>` for modals, `popover`, native form validation, `:has()` and
`:focus-within` for state-dependent styling, CSS scroll-snap, `loading="lazy"` for
images. A controller that reimplements one of these is code you now maintain, with
worse keyboard and screen-reader behavior than the built-in.

Stimulus is right where the platform stops: coordinating several elements,
persisting a choice, debouncing, or wrapping a library.

## Turbo Streams ↔ raw Action Cable

**Question: is the consumer a browser DOM?**

Turbo Streams are HTML operations for a rendered page. If the recipient is another
service, a native app, a printer, a queue worker — anything that wants data rather
than DOM — that's a plain Action Cable channel or a normal API endpoint. Sending
`<turbo-stream>` markup to a consumer with no DOM to apply it to is a category
error, even though both ride the same Action Cable transport.

## Hotwire ↔ a client framework

Hotwire's cost is a round-trip per meaningful change. That's the right trade
almost everywhere and the wrong one when:

- state must survive many interactions without the server (canvas, editors,
  spreadsheets with live recalculation)
- the app must work offline
- interaction runs at animation frequency — drag previews, live cursors
- a large dataset must be filtered/sorted client-side without round-trips

Note what is *not* on that list: "it feels too slow" (measure first — a frame
round-trip is usually tens of milliseconds), "the JS is getting complicated"
(usually means rendering moved into JS and should move back), and "we need
real-time" (that's what broadcasts are).

The supported middle ground is a real component mounted as a leaf inside a
Stimulus controller — `connect()` builds it, `disconnect()` tears it down. Adopting
a client framework as the *application* architecture because one widget needed it
is the thing to push back on.

## Quick table

| Signal | Reach for |
|---|---|
| User is going somewhere new | Drive (plain link) |
| One region navigates, rest of page must hold its state | Frame |
| Region needs its own linkable URL | Frame + `data-turbo-action="advance"` |
| One action changes several disjoint regions | Stream |
| Server-initiated, other users must see it | Broadcast stream |
| Broad change, or markup differs per viewer | `broadcasts_refreshes` + morph |
| Element only exists to receive a stream | Plain element with an id — not a frame |
| Behavior with no server opinion | Stimulus |
| Browser already implements it | Plain HTML/CSS |
| Consumer isn't a DOM | Action Cable channel or API |
| Sustained client state, offline, animation-rate | A real client framework, scoped as a leaf |
