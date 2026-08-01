# Turbo Frames

A frame is a scoping decision: everything inside it navigates inside it. That one
rule explains all frame behavior, including all the ways frames surprise people.

## Contents

- [The matching rule](#the-matching-rule)
- [Targeting and breaking out](#targeting-and-breaking-out)
- [Eager and lazy loading](#eager-and-lazy-loading)
- [Inline edit](#inline-edit)
- [Modals](#modals)
- [Frame + URL](#frame--url)
- [frame-missing](#frame-missing)
- [Styling the busy state](#styling-the-busy-state)
- [When a frame is the wrong tool](#when-a-frame-is-the-wrong-tool)

## The matching rule

When a link or form inside `<turbo-frame id="x">` navigates, Turbo fetches the
response, looks for `<turbo-frame id="x">` **in that response**, and replaces the
frame's contents with the matching frame's contents. Everything else in the
response is discarded.

Consequences worth internalizing:

- The destination page must render the same frame id. A frame that works from the
  index but not the show page usually means the show template forgot the frame.
- The server renders a *whole page* either way — the frame is a client-side
  extraction. `turbo_frame_request?` lets you skip the layout or render a leaner
  template, but never rely on the client to hide expensive work.
- Nested frames are matched by id, not by nesting position.

```erb
<%= turbo_frame_tag "comments" do %>
  <%= render @comments %>
<% end %>

<%# equivalently, with a record: id becomes "comment_42" %>
<%= turbo_frame_tag comment do %>…<% end %>
```

turbo-rails renders frame requests with the `turbo_rails/frame` layout
automatically, so you rarely need `layout: false` by hand.

## Targeting and breaking out

`data-turbo-frame` on a link or form overrides which frame receives the response:

- `data-turbo-frame="_top"` — replace the whole page instead of the frame. Use it
  on "view full record" links inside a frame.
- `data-turbo-frame="other_frame_id"` — drive a *different* frame from outside or
  inside a frame. This is how a filter form outside a results frame updates it.
- `target="_top"` on the `<turbo-frame>` itself — the frame's contents navigate
  the whole page by default, and individual links can opt back in.

```erb
<%# a search form outside the frame that updates it %>
<%= form_with url: posts_path, method: :get, data: { turbo_frame: "results" } do |f| %>
  <%= f.search_field :q %>
<% end %>

<%= turbo_frame_tag "results" do %>
  <%= render @posts %>
<% end %>
```

## Eager and lazy loading

```erb
<%= turbo_frame_tag "stats", src: dashboard_stats_path do %>
  <p>Loading…</p>
<% end %>

<%= turbo_frame_tag "comments", src: post_comments_path(@post), loading: :lazy do %>
  <p>Loading…</p>
<% end %>
```

`src` fetches on connect. `loading="lazy"` defers until the frame scrolls into
view (or becomes visible — so a frame inside a hidden modal loads when the modal
opens, which is the standard modal trick).

This is the cleanest available answer to "this one panel is slow and blocks the
page": move it into a lazy frame and the main page renders immediately. It also
sidesteps a lot of caching complexity — the slow fragment gets its own endpoint
with its own cache headers.

## Inline edit

The canonical frame use, and the one to reach for before any Stimulus:

```erb
<%# posts/_post.html.erb %>
<%= turbo_frame_tag post do %>
  <h2><%= post.title %></h2>
  <%= link_to "Edit", edit_post_path(post) %>
<% end %>

<%# posts/edit.html.erb — same frame id, so it swaps in place %>
<%= turbo_frame_tag @post do %>
  <%= form_with model: @post do |f| %>
    <%= f.text_field :title %>
    <%= f.submit %>
    <%= link_to "Cancel", post_path(@post) %>
  <% end %>
<% end %>
```

`update` then redirects to `post_path(@post)`, whose show template renders the same
frame — the form is replaced by the display markup. No JavaScript at all, and the
edit URL still works as a standalone page.

## Modals

Frame + lazy loading, with the dialog element doing the presentation:

```erb
<%= turbo_frame_tag "modal" %>

<%# link anywhere on the page %>
<%= link_to "New post", new_post_path, data: { turbo_frame: "modal" } %>

<%# posts/new.html.erb %>
<%= turbo_frame_tag "modal" do %>
  <dialog open data-controller="modal">
    <%= render "form", post: @post %>
  </dialog>
<% end %>
```

The Stimulus controller here is doing only presentation (close on Escape, backdrop
click) — the content still comes from the server, and `/posts/new` remains a real
page. That split is the pattern to aim for generally.

## Frame + URL

By default a frame navigation does not change the browser URL. Add
`data-turbo-action="advance"` to the frame or the link to push a history entry, so
the frame's state is shareable and back-button-able:

```erb
<%= turbo_frame_tag "results", data: { turbo_action: "advance" } do %>…<% end %>
```

Use it for tabs, pagination, and filters — anything a user would reasonably expect
to be able to link to. Skip it for inline edit and modals.

## frame-missing

If the response has no matching frame, Turbo's default handling is **loud, not
silent** — verify this in your own bundled version before reasoning from it, but
in Turbo 8 the frame's `view.missing()` replaces its contents with
`<strong class="turbo-frame-error">Content missing</strong>` and then throws
`TurboFrameMissingError`, whose message names `turbo-visit-control` as the fix.

That matters for diagnosis: **"the frame did nothing at all, and the console is
clean" is evidence against frame-missing**, not for it. A frame that hit this path
shows the error text. If the frame is genuinely inert with nothing logged, look
instead at whether a request was ever made — `data-turbo="false"` on an ancestor,
a `data-turbo-preview` snapshot the frame is inert inside, an already-`complete`
frame, or JavaScript that died earlier on the page.

Common causes once you've confirmed it *is* frame-missing, in order of likelihood:

1. The destination template doesn't render the frame.
2. A redirect landed somewhere that doesn't render it — sign-in pages are the
   classic case: the session times out, the login page has no `comments` frame,
   and the frame silently dies.
3. The response was an error page.

Handle it deliberately when the target may legitimately be a full page:

```js
document.addEventListener("turbo:frame-missing", (event) => {
  event.preventDefault()
  event.detail.visit(event.detail.response) // fall back to a full-page visit
})
```

Server-side, `<meta name="turbo-visit-control" content="reload">` in the response
forces the whole page to load instead — the standard fix for the login redirect.

## Styling the busy state

Turbo sets `[aria-busy="true"]` on a navigating frame and `[complete]` when done.
That's a free loading indicator with no JavaScript:

```css
turbo-frame[aria-busy="true"] { opacity: .6; cursor: progress; }
```

## When a frame is the wrong tool

- The action must update **two or more** disjoint regions (list *and* counter *and*
  flash) — use a stream.
- The update must reach other users — use a broadcast stream.
- The content isn't a navigation at all (toggling a class) — use Stimulus.
- You are nesting frames three deep to express a layout — the page probably wants
  restructuring, not more frames.
