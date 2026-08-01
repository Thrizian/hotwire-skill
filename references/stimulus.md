# Stimulus

Stimulus attaches behavior to HTML the server already rendered. It does not own
the DOM, does not render, and holds no application state. The HTML is the source
of truth; the controller reads it and reacts to it.

That framing is the whole discipline. A controller that builds markup from a JSON
response is fighting the architecture — that markup should have come from the
server.

## Contents

- [Controller shape](#controller-shape)
- [Targets](#targets)
- [Values](#values)
- [Classes](#classes)
- [Outlets](#outlets)
- [Actions](#actions)
- [Lifecycle](#lifecycle)
- [Controller-to-controller communication](#controller-to-controller-communication)
- [Turbo interop](#turbo-interop)
- [Anti-patterns](#anti-patterns)

## Controller shape

`app/javascript/controllers/clipboard_controller.js` → identifier `clipboard`.
Multi-word: `date_picker_controller.js` → `data-controller="date-picker"`.

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source", "button"]
  static values = { successDuration: { type: Number, default: 2000 } }
  static classes = ["copied"]

  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value)
    this.element.classList.add(this.copiedClass)
    setTimeout(() => this.element.classList.remove(this.copiedClass),
               this.successDurationValue)
  }
}
```

```erb
<div data-controller="clipboard"
     data-clipboard-copied-class="bg-green-100"
     data-clipboard-success-duration-value="1500">
  <input data-clipboard-target="source" value="…" readonly>
  <button data-action="clipboard#copy">Copy</button>
</div>
```

An element can carry several controllers: `data-controller="clipboard tooltip"`.
Small, composable controllers beat one big one — a `toggle` controller reused in
ten places is worth more than ten bespoke ones.

## Targets

```js
static targets = ["query", "result"]
```

- `this.queryTarget` — first match; **throws** if absent.
- `this.queryTargets` — array.
- `this.hasQueryTarget` — boolean; check this for optional targets.
- `queryTargetConnected(el)` / `queryTargetDisconnected(el)` — fire as elements
  enter/leave, including when Turbo Streams inject them. This is the correct hook
  for "initialize each new row that arrives" — not a MutationObserver.

Targets are scoped to the controller element's subtree, and a nested controller of
the same identifier claims its own.

## Values

```js
static values = {
  url: String,
  refreshInterval: { type: Number, default: 5000 },
  filters: Array,
  open: Boolean
}
```

Types: `Array`, `Boolean`, `Number`, `Object`, `String`. Defaults when the
attribute is missing: `[]`, `false`, `0`, `{}`, `""`.

**Every piece of server state a controller reads is a value.** The moment you
find yourself writing `element.dataset.somethingCustom`, `getAttribute("data-…")`
or a marker attribute the server sets for the controller to find — stop, that is
a value you have not declared. Declaring it gets you typing, a default, a
`hasXValue` guard, a change callback, and one name instead of two spellings that
drift apart. The hand-rolled version gets you none of that and reads as though
the author didn't know the API:

```js
// No. A private protocol between one template and one controller.
const selected = this.tabTargets.find(tab => tab.dataset.tabsAskedFor === "true")

// Yes. Declared, typed, guarded, and greppable from both sides.
static values = { selected: String }   // data-tabs-selected-value="…"
```

Targets are for *elements the controller operates on*; values are for *data it
reads*. A marker attribute on a target is nearly always a value on the
controller element wearing the wrong hat.

Attribute naming is camelCase → kebab-case:
`refreshInterval` → `data-[identifier]-refresh-interval-value`.

Each value gets `this.xValue` (read), `this.xValue = …` (write, which updates the
DOM attribute), and `this.hasXValue`.

`xValueChanged(value, previousValue)` fires once at connect and on every change.
Two consequences to design around: it runs *before* `connect()`, so don't touch
targets in it without a `hasXTarget` guard; and because writing a value updates the
attribute, values are how you keep state *in the DOM* where Turbo can see it.

```js
openValueChanged(open) {
  this.element.toggleAttribute("hidden", !open)
}
```

## Classes

Keep CSS class names out of JavaScript so designers can change them in the markup:

```js
static classes = ["loading", "error"]
// this.loadingClass  (data-[identifier]-loading-class="…")
// this.loadingClasses (space-separated list)
// this.hasLoadingClass
```

## Outlets

References to controllers *elsewhere on the page*, not inside this controller's
subtree.

```js
static outlets = ["result-list"]
// data-search-result-list-outlet=".results"
```

Gives `this.resultListOutlet` (controller instance), `…Outlets`, `…OutletElement`,
`…OutletElements`, `hasResultListOutlet`, plus
`resultListOutletConnected(controller, element)` / `…Disconnected`.

Outlets couple two controllers together. Prefer `dispatch` unless the caller
genuinely needs to *call methods* on the other controller.

## Actions

`data-action="event->identifier#method"`, space-separated for several, invoked
left-to-right.

Default events by element, letting you omit `event->`:

| Element | Default |
|---|---|
| `a`, `button`, `input[type=submit]` | `click` |
| `form` | `submit` |
| `input`, `textarea` | `input` |
| `select` | `change` |
| `details` | `toggle` |

Options, appended after the method: `:once`, `:capture`, `:passive`, `:!passive`,
`:stop` (stopPropagation), `:prevent` (preventDefault), `:self` (ignore events from
descendants).

Keyboard filters: `keydown.esc->modal#close`, also `enter`, `tab`, `space`, arrows,
`home`, `end`, `page_up`, `page_down`, `[a-z]`, `[0-9]`, and modifier combos like
`ctrl+a`, `shift+enter`.

Global targets: `data-action="resize@window->gallery#layout"`, `@document` likewise.
These are unbound automatically on disconnect — which is exactly why they beat a
hand-written `addEventListener` in `connect()`.

Action params pass data without inventing attributes:

```erb
<button data-action="cart#add" data-cart-item-id-param="42" data-cart-qty-param="2">
```

```js
add({ params: { itemId, qty } }) { … }
```

## Lifecycle

- `initialize()` — once per controller instance, before first connect.
- `connect()` — every time the element enters the DOM *and* on page load. Must be
  safe to run repeatedly: Turbo Drive renders, frame loads, and stream inserts all
  trigger it.
- `disconnect()` — element leaves. Tear down here what you built in `connect()`:
  timers, observers, third-party instances. Anything you set up with
  `addEventListener` on window/document must be removed here or it leaks and
  double-fires after a few navigations.
- Static `shouldLoad()` gates registration; `afterLoad()` runs once after it.

```js
connect() {
  this.timer = setInterval(() => this.refresh(), this.refreshIntervalValue)
}
disconnect() {
  clearInterval(this.timer)
}
```

## Controller-to-controller communication

`dispatch` emits a CustomEvent namespaced by identifier:

```js
this.dispatch("selected", { detail: { id: 42 } })  // → "search:selected"
```

```erb
<div data-controller="results" data-action="search:selected->results#highlight"></div>
```

Prefer this over outlets and far over reaching into another controller with
`getControllerForElementAndIdentifier` (documented as a last resort).

## Turbo interop

- **Never bind `turbo:load` to set up per-element behavior.** It fires on every
  visit and re-binds listeners to elements that already have them — the classic
  "my handler fires three times" bug. Use a controller; `connect()` already gives
  you the right lifecycle.
- **Turbo cache previews:** `connect()` can run against a snapshot that is about to
  be discarded. Guard side-effecting setup:
  `if (document.documentElement.hasAttribute("data-turbo-preview")) return`.
- **If a widget mutates the markup, `disconnect()` is too late.** Turbo caches the
  page *before* the next one renders — `cacheSnapshot()` fires `turbo:before-cache`,
  awaits one event-loop tick, then calls `snapshot.clone()` — while Stimulus only
  disconnects once the incoming page renders, after that clone. The snapshot
  therefore freezes whatever the widget did to the DOM. Tear those down on
  `turbo:before-cache`, and keep `disconnect()` for what it does cover: the element
  being removed by a frame load or a stream.

  This is why wrapped date pickers, selects and editors come back inert after a
  Back navigation. flatpickr sets `input.type = "text"` and stashes the original on
  `input._type` — a JavaScript property, not an attribute — so `cloneNode` drops it
  and the restored page has a plain text field that can't be restored.

  ```js
  connect() {
    this.picker = flatpickr(this.element, {})
    this.teardown = () => this.picker?.destroy()
    document.addEventListener("turbo:before-cache", this.teardown)
  }

  disconnect() {
    document.removeEventListener("turbo:before-cache", this.teardown)
    this.picker?.destroy()
  }
  ```
- **Morphing** keeps existing elements, so `connect()` does *not* re-run after a
  page refresh. Don't put "recompute after every update" logic there; use a value
  change callback or a target-connected callback instead.
- **Streams** insert elements after load; `targetConnected` and `connect` both fire
  normally. No extra wiring needed.
- **Third-party widgets** belong in `connect()`/`disconnect()` pairs, and their
  container usually wants `data-turbo-permanent` so navigation doesn't thrash them.

## Anti-patterns

- Fetching JSON and building DOM in a controller → return HTML and use a frame or
  stream.
- Storing application state in controller instance fields that must survive
  navigation → put it in the DOM as a value, or on the server.
- One 400-line controller per page → split by behavior and compose on the element.
- `document.querySelector` inside a controller → that's what targets and outlets
  are for; querying outside your scope makes the controller un-reusable.
- `addEventListener` in `connect()` without the matching `removeEventListener` in
  `disconnect()` → use `data-action` with `@window`/`@document` instead.
