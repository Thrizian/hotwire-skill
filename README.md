# hotwire 

Expert guidance for building Rails apps with [Hotwire](https://hotwired.dev):
Turbo Drive, Turbo Frames, Turbo Streams and Stimulus.

## Why this exists

Coding agents reach for JavaScript. Ask for a filtered list and you get a fetch
call, a JSON endpoint and a template string; ask for an inline edit and you get a
controller that hides one div and shows another. It is all competent code, and it
is all a second copy of your rendering layer written in the wrong language — the
exact thing Hotwire exists to delete.

The pull is structural, not accidental: the training data is overwhelmingly
client-rendered, so "add interactivity" resolves to "write JavaScript" long before
anyone considers that the server could have sent the HTML. Hotwire is a small DSL
with strong opinions, and an agent that doesn't know those opinions will route
around them while producing something that works.

So this skill is mostly a ladder — plain HTML, Drive, Frame, Stream, Stimulus —
and the discipline of taking the lowest rung that solves the problem, with the
boundaries between the rungs written down and sourced. Stimulus is the top rung,
not the default one. It attaches behavior to server-rendered HTML; it is not a
rendering layer, and a controller that turns JSON into DOM is the failure this
skill is built to catch.

## Install

```bash
git clone https://github.com/Thrizian/hotwire-skill.git ~/.claude/skills/hotwire
```

Claude Code picks it up from `~/.claude/skills/`. It triggers on Hotwire work
even when Turbo and Stimulus are never named — inline edit, tabs, live updates,
"without a full page reload", or reaching for React to do something a
server-rendered response could do.

## What's in it

| File | Covers |
|---|---|
| `SKILL.md` | The ladder, how to shape an answer, checking the host project before writing |
| `references/choosing.md` | The boundaries — Drive ↔ Frame ↔ Stream ↔ Stimulus, and where Hotwire hands off |
| `references/turbo-drive.md` | Navigation, snapshot cache and previews, prefetch, asset tracking, morphing, events |
| `references/turbo-frames.md` | Frame matching, targeting, lazy loading, inline edit, modals, frame-missing |
| `references/turbo-streams.md` | The nine actions, broadcasting, `broadcasts_refreshes`, when not to use them |
| `references/stimulus.md` | Targets, values, classes, outlets, actions, lifecycle, Turbo interop |
| `references/debugging.md` | Symptom-first decision trees for the quiet failures |

Reference files load only when relevant, so the always-resident cost stays small.

## Three principles it enforces

**Answer with the change, not a lecture.** Lead with the edit and its file path.
A diagnosis is one or two sentences; four ranked hypotheses and a testing
appendix are not an answer. Code reviews are the deliberate exception — there,
brevity means one line per finding, never fewer findings.

**Check how the app actually renders before writing a line.** Examples here are
ERB because a portable skill has to be; the project wins on every such detail.
Read a partial's contract before calling it.

**Don't assert what you haven't checked.** Read the installed source rather than
trusting docs or memory, keep the seam visible between what you read and what you
inferred, and name the claims you couldn't verify.

## Corrections this skill carries

Things widely written about Hotwire that turn out to be wrong against the
installed source (verified on Turbo 8.0.23, Stimulus 3.2.2, turbo-rails 2.0.23):

- **Frame-missing is not silent.** Turbo paints
  `<strong class="turbo-frame-error">Content missing</strong>` into the frame and
  throws `TurboFrameMissingError`. So "the frame did nothing and the console is
  clean" is evidence *against* the usual session-expiry diagnosis — look for a
  request that never fired.
- **`disconnect()` is too late** to keep a markup-mutating widget out of the Turbo
  snapshot. `cacheSnapshot()` fires `turbo:before-cache`, awaits a tick, then
  clones — while Stimulus disconnects only when the next page renders. Tear such
  widgets down on `turbo:before-cache`.
- **The tracked-asset check is Drive-only.** It lives in `PageRenderer.shouldRender`,
  so frame loads and stream responses bypass it — a long-lived page updated purely
  over streams can run a stale bundle indefinitely.

## How it was built

Written against primary sources, then revised across three eval iterations plus a
boundary-triage run, each graded by independent agents that verified every
citation against a real codebase.

- Answer quality vs. no skill: **26/27 → 21/27**
- Boundary triage across ten Drive/Frame/Stream/Stimulus decisions: **30/30**
- Trigger accuracy: **80% recall, 0% false positives** across twenty queries,
  half of them deliberately adjacent (mailer HTML, `pg_search` relevance, CSV
  export, machine-to-machine WebSockets, service workers)

`evals/evals.json` holds the original three task evals.

Two known triggering gaps, structural rather than fixable by wording: requests
that read as one small step ("add a confirm dialog", "make this accordion
expand") get answered directly without consulting any skill, even when they name
Stimulus outright.
