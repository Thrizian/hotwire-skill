# Turbo Streams

A stream is a list of DOM operations addressed by id. The server says "append this
markup to `#comments` and replace `#comment_count`", and Turbo does exactly that.
Streams are deliberately limited to nine actions — anything cleverer belongs in a
Stimulus controller attached to the markup you send.

## Contents

- [The nine actions](#the-nine-actions)
- [Targeting](#targeting)
- [Responding from a controller](#responding-from-a-controller)
- [Broadcasting from models](#broadcasting-from-models)
- [broadcasts_refreshes](#broadcasts_refreshes)
- [Custom stream actions](#custom-stream-actions)
- [Design rules](#design-rules)

## The nine actions

Every action except `remove` and `refresh` wraps its content in `<template>`.

| Action | Effect |
|---|---|
| `append` | add to end of target's children |
| `prepend` | add to start of target's children |
| `before` | insert as sibling before target |
| `after` | insert as sibling after target |
| `replace` | replace the target element itself |
| `update` | replace the target's *inner* content, keeping the element |
| `remove` | delete the target |
| `morph` | morph the target toward the new content instead of swapping |
| `refresh` | trigger a page refresh (carries a `request-id` so the initiator skips it) |

`replace` vs `update` matters more than it looks: `replace` destroys the element,
so any Stimulus controller on it disconnects and reconnects and any client state on
it is lost. `update` keeps the element. Reach for `update` when the container
carries a controller or scroll position.

Raw wire format:

```html
<turbo-stream action="append" target="comments">
  <template><div id="comment_42">…</div></template>
</turbo-stream>
```

## Targeting

- `target="dom_id"` — a single element by id.
- `targets=".css .selector"` — every element matching a CSS selector.

Rails helpers accept a record and call `dom_id` for you, which is why consistent
`dom_id` use in partials is load-bearing:

```ruby
turbo_stream.append :comments, partial: "comments/comment", locals: { comment: @comment }
turbo_stream.replace @comment                      # target = dom_id(@comment)
turbo_stream.remove @comment
turbo_stream.update :comment_count, @post.comments.size
turbo_stream.append_all ".comment-list", partial: "…"
```

## Responding from a controller

Two shapes. Inline for one or two operations:

```ruby
def destroy
  @comment.destroy
  respond_to do |format|
    format.turbo_stream { render turbo_stream: turbo_stream.remove(@comment) }
    format.html { redirect_to @post }
  end
end
```

A template when there are several — `app/views/comments/create.turbo_stream.erb`:

```erb
<%= turbo_stream.append "comments" do %>
  <%= render @comment %>
<% end %>
<%= turbo_stream.update "comment_count", @post.comments.size %>
<%= turbo_stream.replace "new_comment_form" do %>
  <%= render "form", comment: Comment.new %>
<% end %>
```

Keep the `format.html` branch. It is what makes the flow testable without a
browser and what saves you when a client sends a plain request.

Multiple operations can also be returned from an inline render by passing an array:

```ruby
render turbo_stream: [
  turbo_stream.append("comments", @comment),
  turbo_stream.update("comment_count", @post.comments.size)
]
```

## Broadcasting from models

For updates that must reach *other* users' open pages, over Action Cable.

Subscribe in the view:

```erb
<%= turbo_stream_from @post %>
```

Broadcast from the model:

```ruby
class Comment < ApplicationRecord
  belongs_to :post

  after_create_commit  -> { broadcast_append_later_to post, target: "comments" }
  after_update_commit  -> { broadcast_replace_later_to post }
  after_destroy_commit -> { broadcast_remove_to post }
end
```

Or declaratively, which wires create/update/destroy at once:

```ruby
broadcasts_to ->(comment) { comment.post }
```

Rules that save real debugging time:

- Use the `_later_to` variants for create/update. They render the partial in a
  background job, off the request. `broadcast_remove_to` has no `_later` need —
  there's nothing to render.
- Broadcast from `after_*_commit`, never `after_save`. Broadcasting inside the
  transaction can render a record readers can't see yet.
- `<turbo-stream-source>` (what `turbo_stream_from` emits) must live in `<body>`,
  not `<head>` — and not inside `data-turbo-permanent`, or it won't reconnect.
- Broadcast partials render **without a request**: no `current_user`, no session,
  no request-dependent helpers. If the markup differs per viewer, you cannot
  broadcast one payload to everyone — broadcast a refresh instead, or broadcast to
  a per-user stream.
- Development's default Action Cable adapter is `async`, single-process. Broadcasts
  from a background job in a separate process won't arrive. Use `redis` when
  testing broadcasts for real.
- WebSockets drop. A user who missed a broadcast sees stale content until they
  navigate. Design so that's merely stale, not broken.

## broadcasts_refreshes

Turbo 8's answer to the per-attribute broadcast sprawl above:

```ruby
class Post < ApplicationRecord
  broadcasts_refreshes
end
```

Every change broadcasts a debounced `refresh` stream; each subscribed client
re-requests the page and morphs it. Per-viewer differences resolve themselves
because each client renders its own request.

Cost: a full page render per client per change. Good for pages that change
occasionally in many places; bad for a high-frequency ticker where a targeted
`update` on one span is far cheaper.

## Custom stream actions

The nine actions are deliberate. When you genuinely need a new primitive
(scroll-into-view, toast, redirect), register one rather than smuggling `<script>`
into a template:

```js
import { StreamActions } from "@hotwired/turbo"

StreamActions.scroll_to = function () {
  document.getElementById(this.target)?.scrollIntoView({ behavior: "smooth" })
}
```

```ruby
turbo_stream_action_tag :scroll_to, target: "comment_42"
```

Before doing this, check whether the same effect belongs in a Stimulus controller
that ships *with* the markup you're already appending — usually it does, and it
keeps the behavior discoverable from the HTML.

## Design rules

- **Build without streams first.** Streams are a layer over a flow that already
  works with redirects.
- **One partial, both paths.** The stream and the full page render the same partial
  or they will drift.
- **Don't stream what a frame can do.** One region responding to one navigation is
  a frame; multiple regions or other users is a stream.
- **Don't put `<script>` in stream content.** Attach behavior with
  `data-controller` on the markup you send.
- **Keep ids stable and derived.** `dom_id(record)` in the partial, the same id in
  the stream target. A stream targeting a missing id is a silent no-op — the single
  most common "my stream doesn't work" cause.
