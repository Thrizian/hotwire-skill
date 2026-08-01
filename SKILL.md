---
name: hotwire
description: Expert guidance for building with Hotwire — Turbo Drive, Turbo Frames, Turbo Streams, and Stimulus — sending HTML over the wire instead of JSON. Use this skill when the user works on turbo frames or streams, Stimulus controllers, model broadcasts, or page refresh/morphing, and when debugging why a frame won't navigate, a stream won't apply, or a Stimulus controller won't connect. Use it just as readily for any of the following even when Hotwire, Turbo and Stimulus are never named: updating part of a page without a full reload; inline edit in place; modals or dialogs whose content comes from the server; tabs; filtering, sorting, searching or paginating a list without losing scroll position or while keeping the URL shareable; lazy-loading a slow section so the rest of the page renders first; live, real-time or auto-refreshing content that other people's open pages should see; form submissions whose validation errors must appear without losing the page; confirmation dialogs on links or buttons, including custom-styled ones; infinite scroll; toggling, dropdowns, debouncing or any other JavaScript behavior added to a server-rendered Rails view; users running stale JavaScript or CSS after a deploy until they hard-refresh, or any long-lived tab drifting out of sync with the server; or the user reaching for React, Vue, fetch+JSON or hand-written AJAX to do something a server-rendered HTML response could do.
---

# Hotwire

Hotwire's premise: the server already knows how to render HTML, so send HTML.
The client's job shrinks to swapping it in. That is why Hotwire apps stay small —
there is no second copy of the domain model living in JavaScript, no serializer
layer, no client router, no cache invalidation problem in two places.

The main failure mode when working with Hotwire is not writing wrong code — it is
reaching for a heavier tool than the problem needs. Someone writes a Stimulus
controller that fetches JSON and builds DOM, when a Turbo Frame would have done
it in two lines of HTML. Most of the value you add here is pushing work *down*
this ladder.

## Answer with the change, not a lecture

The value you deliver is the edit to the file, not the tour of the framework. Lead
with the code that changes, say which file it goes in, and stop. Someone asking
why their frame is dead wants the one-line cause and the fix — not four ranked
hypotheses, a debugging methodology, and a testing appendix they didn't ask for.

Hold to this shape:

- **Diagnosis or decision: one or two sentences.** "Session expired; the login
  redirect has no matching frame, so Turbo drops the response." That's the whole
  explanation. The mechanism is only worth more words if they can't act without it.
- **Then the files.** Each with its real path, containing only the lines that
  change. Complete enough to paste, short enough to read.
- **Trailing caveats: at most two, one line each**, and only for things that will
  actually bite — the 422, the `after_create_commit`, the dev Action Cable adapter.
- **Alternatives you considered: normally zero.** Pick one and say why in a clause.
  A menu is work handed back to the person who asked.

What to leave out unless asked: framework background they already have, the full
debugging decision tree (you read it, you don't recite it), test suites for code
they haven't merged, sections labelled "Further improvements", and restating the
diff in prose underneath the diff.

**Short is not a licence to guess.** Cutting the four-hypothesis essay means doing
the narrowing yourself, not skipping it. Before you name a cause, go and look:
open the sign-in layout and see whether it renders the frame, grep for the frame
id in both templates, check the Devise timeout setting, read the controller's
`before_action`s, look at the actual response in the log. Those are seconds of
work and they turn a plausible story into a finding.

Then say what you found, not what you suspect: "`app/views/devise/sessions/new.html.haml`
has no `users` frame, so the timeout redirect gets discarded" beats "this is
probably a session issue" — same length, and one of them is checked. When you
genuinely cannot verify something (no repo access, needs a browser, needs their
logs), say which claim is unverified and name the single check that settles it.
An honest "I can't see X from here; run this and tell me" is worth more than a
confident paragraph that happens to be wrong.

**Keep the seam visible between what you read and what you concluded.** Reading a
guard in the source tells you the guard exists; it does not tell you the state
that trips it ever occurs in their app. Those are different claims and the second
one is usually where the error hides. Write it as what it is — "the frame is inert
whenever `data-turbo-preview` is set (source); *if* something leaves the page
stuck in that state, that explains the symptom — check with `…`" — rather than
letting a verified fact carry an unverified story on its back. A reader can act on
a labelled inference; they cannot act on one disguised as a finding.

Small thing with the same shape: read versions from what is installed
(`node_modules/@hotwired/turbo/package.json`, the lockfile), not from the range
declared in `package.json`. `^8.0.12` is a constraint, not a fact about what is
running.

If two causes both survive checking, name the likelier one, give its fix, and add
one line on what would prove it's the other. That's shorter than covering both and
more useful than hedging across both.

Length is a proxy, not the goal, but it's a useful alarm: if the answer is longer
than the code it changes, you're writing a tutorial. Cut it.

**Reviews are the exception, and only in one direction.** When someone asks you to
review code before merging, finding every real defect *is* the deliverable —
dropping the race condition, the missing error handling, the untranslated string
or the unindexed query to hit a word count makes the review worse, not tighter.
Brevity in a review means one line per finding, ordered with the blocking ones
first, no restating the code back at them, no praise. It never means fewer
findings. Say the defect, say why it bites, move on.

## Before you write a line: check how this app renders

Every example below is ERB, because a portable skill has to be readable anywhere.
That is a limitation of this file, not a recommendation. **Look at the project's
existing views before writing any, and write what it writes** — HAML, Slim,
ViewComponent, whatever is there. Handing someone ERB in a HAML codebase is
handing them a translation job, and "adjust to your template language as you go"
is not an acceptable substitute for opening one view file and seeing.

The same goes for everything around the Hotwire part: if the app authorizes with
Pundit, your controller calls `policy_scope`; if it paginates with Kaminari, your
frame paginates with Kaminari; if it renders cards through a ViewComponent, your
stream renders that component rather than a new partial beside it. A technically
correct frame that ignores the surrounding conventions is still a worse answer,
because someone now has to rewrite it before it can merge.

Cheapest possible check, one command:

```bash
ls app/views/**/*.haml app/views/**/*.erb 2>/dev/null | head
```

One caveat that matters more than it sounds: **open any partial, component or
helper before you call it.** Matching the local idiom means using the local pieces
within their actual contract — what locals they require, what ivars they assume,
what they render when those are missing. A partial borrowed from a CRUD scaffold
may quietly depend on `@resource` or a context object your page never sets, and
happen to render blank instead of raising. That is not working code, it's code
that hasn't failed yet. Same rule when you're about to write a file that already
exists: read it first and edit it, rather than emitting a fresh body that silently
replaces someone's work.

## The ladder — always take the lowest rung that works

**0. Plain HTML.** A link, a form, a redirect. Turbo Drive already makes this feel
instant: it intercepts the navigation, fetches the page, swaps `<body>`, and
restores scroll. No code. Start every feature here and only climb when this rung
demonstrably fails.

**1. Turbo Drive tuning.** The page navigation is right, you just want different
mechanics: `data-turbo-action="replace"`, `data-turbo-confirm`, `data-turbo-method`,
prefetch control, or Turbo 8 morphing so a refresh preserves scroll and focus.
See `references/turbo-drive.md`.

**2. Turbo Frame.** *One* region of the page should navigate independently while
the rest stays put — inline edit, a tab panel, pagination, a lazily-loaded
sidebar, a modal. A frame is a scoping decision, not a rendering technique: links
and forms inside it stay inside it. See `references/turbo-frames.md`.

**3. Turbo Stream.** One user action must change *several* places at once, or a
change must reach *other* users' browsers. A stream is a list of DOM operations
(append/replace/remove/…) addressed by target id. Streams are for fan-out; if you
are updating exactly one region in response to one navigation, a frame is simpler
and degrades better. See `references/turbo-streams.md`.

**4. Stimulus.** The behavior has no server round-trip in it at all — toggling a
class, focusing a field, debouncing input, driving a third-party widget,
formatting on keystroke. Stimulus is a way to attach behavior to HTML that the
server sent; it is not a rendering layer. See `references/stimulus.md`.

Whatever the server needs to tell that controller goes through `static values`,
which is the API for exactly this — not a data attribute you invent and read
back with `dataset`. Write the value declaration before the markup, so the
attribute name is derived from it rather than the other way round.

Climbing is easy to justify and hard to undo, so state the rung you picked and why
the one below it doesn't work. If you can't articulate that, you're probably one
rung too high.

When two rungs both look plausible, the discriminator is usually **what triggered
the change**, not what the change looks like on screen: a navigation inside one
region is a frame; state changing in several places at once, or changing without
any navigation, is a stream; behavior with no server opinion at all is Stimulus.
`references/choosing.md` works each boundary with the question that settles it and
an adjacent pair that sits either side of the line — read it whenever you're
deciding between two tools rather than reaching for a known one.

One rule worth carrying without looking it up, because it comes straight from the
handbook and gets violated constantly: **an element that exists only to receive a
stream should not be a `<turbo-frame>`.** Use a plain element with an id. Making it
a frame buys navigation semantics you didn't want and a `turbo:frame-missing`
failure mode you didn't need.

## Progressive enhancement is the design constraint, not a nicety

Build the flow so it works with Turbo entirely absent — real routes, real
redirects, real full-page renders — then layer frames and streams on top. This is
not ceremony. It is what makes the code testable without a browser, keeps URLs
meaningful and shareable, gives you working back-button behavior for free, and
means a dropped WebSocket degrades into a stale page rather than a broken one.

Concretely: a controller action that can *only* answer `turbo_stream` is a smell.
Give it an HTML path too.

```ruby
def create
  @comment = @post.comments.new(comment_params)
  if @comment.save
    respond_to do |format|
      format.turbo_stream # app/views/comments/create.turbo_stream.erb
      format.html { redirect_to @post }
    end
  else
    # 422 matters: Turbo only renders a form response for a 4xx/5xx status
    render :new, status: :unprocessable_entity
  end
end
```

Two Rails-side rules that cause most "my form does nothing" reports:

- A **successful** POST/PATCH/DELETE must redirect (Turbo follows it as a 303 for
  non-GET). Rendering a 200 body after a successful POST leaves Turbo with nothing
  to navigate to.
- A **failed** submit must render with a 4xx status (`:unprocessable_entity`).
  A 200 is treated as "nothing to see here" and your error messages vanish.

## Where the render lives

Frames and streams both re-render *the same partials* the full page uses. If you
find yourself writing markup that exists only for the stream response, extract a
partial and use it in both places — the whole economy of Hotwire comes from that
partial being the single source of truth for how a thing looks.

```erb
<%# app/views/comments/_comment.html.erb — used by index, by the frame, by the stream %>
<div id="<%= dom_id(comment) %>" class="comment">
  <%= comment.body %>
</div>
```

`dom_id` matters more than it looks: streams and frames address elements by id, so
consistent ids are the addressing scheme of the whole system.

**Derive every id, including the ones that aren't a record.** `dom_id` takes a
prefix, so a heading, an empty state, and a form all get ids tied to the record
they belong to:

```erb
<%# ERB %>
<div id="<%= dom_id(@post, :comments) %>">…</div>
<h2  id="<%= dom_id(@post, :comments_heading) %>">…</h2>
```

```haml
-# HAML — write the native syntax, not the tag helper
.comments{ id: dom_id(@post, :comments) }
  = render @comments
%h2{ id: dom_id(@post, :comments_heading) }
  = t(".count", count: @post.comments.size)
```

Reach for `tag.div`/`content_tag` only where you actually need to build markup in
Ruby. Inside a template, use that template language's own element syntax — a HAML
file full of `= tag.div` is ERB written in the wrong file, and it reads as though
whoever wrote it didn't look at the surrounding views.

```erb
<%= turbo_stream.append dom_id(@post, :comments) do %>…<% end %>
<%= turbo_stream.replace dom_id(@post, :comments_heading) do %>…<% end %>
```

The alternative — `"comments"`, `"comments_heading"` written as string literals in
the view and again in the stream template — is the same id maintained in two
files with nothing connecting them. When one moves, the stream silently targets
nothing: no error, no change on screen, and a bug that only shows up in a browser.
Deriving both sides from the same call makes that class of failure impossible, and
it scales to more than one post on a page for free.

## When Hotwire is the wrong answer

Be straight about this rather than evangelizing. Hotwire loses when the
interaction has **client-side state that must survive many interactions without
the server**, or when latency per interaction is unacceptable:

- Canvas/WebGL, video editors, drawing tools, spreadsheets with live formula recalc
- Offline-first apps, or anything that must keep working with no connection
- Interactions at animation frequency — drag-to-reorder previews, live cursors
- Very large client-side data grids with sorting/filtering that must not round-trip

Hotwire wins for the ordinary majority: CRUD, forms, dashboards, filters, search,
comment threads, live-updating lists, wizards, modals, nested forms.

The middle ground worth naming: you can drop a real component (React, a chart lib,
a rich text editor) into a Stimulus controller as a leaf. That's a supported
shape, not a defeat — Stimulus mounts it in `connect()` and tears it down in
`disconnect()`. What you should push back on is adopting a client framework as the
*application architecture* because one widget needed it.

If the user is already building JSON endpoints plus fetch calls to update a page
region, say plainly what it would look like as a frame or stream and let them
choose. Show the smaller version rather than arguing about it in the abstract.

## Debugging

Hotwire failures are quiet by design — a frame that can't find its match, a stream
targeting a missing id, a controller whose identifier doesn't match the filename.
Nothing throws; the page just sits there. When something "doesn't work", read
`references/debugging.md` and work the decision tree rather than guessing. It
covers frame-missing, streams that arrive but don't apply, double-firing event
listeners after Turbo cache restores, controllers that never connect, and
morphing that eats client state.

## Turbo 8 morphing — worth knowing before you reach for streams

Turbo 8 can refresh the current page and morph only the changed DOM nodes,
preserving scroll and focus. That collapses a whole class of problems that used to
need hand-written streams: after any change, broadcast a refresh and let every
client re-render itself from the truth.

```ruby
class Post < ApplicationRecord
  broadcasts_refreshes # debounced refresh broadcast to subscribers
end
```

```erb
<%# in the layout or view %>
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
<%= turbo_stream_from @post %>
```

Trade-off to state honestly: morphing re-renders the whole page server-side, so
it's more server work per update than a targeted stream, and it can disturb
un-marked client state (mark it `data-turbo-permanent`). Prefer it when the update
touches many parts of the page or when hand-maintaining stream targets is getting
fiddly; prefer targeted streams for high-frequency updates to one small region.

## Testing

Prefer request specs / integration tests over browser tests wherever the response
is server-rendered: assert the frame id is present, assert the turbo_stream
response contains the expected `<turbo-stream action=… target=…>`, assert the
failed-submit status is 422. Reserve real-browser tests for behavior that only
exists in the browser — Stimulus interactions and broadcast delivery.

```ruby
post post_comments_path(post), params: { comment: { body: "hi" } },
     as: :turbo_stream
expect(response.media_type).to eq Mime[:turbo_stream]
expect(response.body).to include "turbo-stream action=\"append\" target=\"comments\""
```

## Reference files

Read the one that matches what you're doing; don't read all four.

- `references/choosing.md` — the boundaries between Drive/Frame/Stream/Stimulus,
  plus where Hotwire hands off to the platform, to Action Cable, and to a real
  client framework. Read this when two tools both look plausible.
- `references/turbo-drive.md` — navigation, cache and preview, prefetch/preload,
  form + redirect rules, page refresh & morphing, the full event list
- `references/turbo-frames.md` — frame matching rules, targeting and breaking out,
  lazy loading, modals, inline edit, frame-missing
- `references/turbo-streams.md` — all nine actions with exact markup, responding
  from controllers, model broadcasting, Action Cable notes, when not to use them
- `references/stimulus.md` — controllers, targets, values, classes, outlets,
  action syntax and options, lifecycle, Turbo interop patterns
- `references/debugging.md` — symptom-first decision trees
