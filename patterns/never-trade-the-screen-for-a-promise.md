---
type: pattern
date: "2026-08-19"
source: The dashboard PR #6 — the first outside contribution, authored by the first outside tester's own agent (Windows, day one)
tags:
  - ui
  - state
  - drift
  - anti-pattern
---

# Never Trade the Screen for a Promise

The classic refresh shape — clear the view, then fetch its replacement — makes
a silent bet: that the fetch will succeed. When it doesn't, the member's state
isn't refreshed, it's *destroyed*. The report that found this read "as I was
scrolling, the whole chat disappeared": a finished turn cleared the pane
synchronously, the history read failed, and a live conversation became a blank
screen. Scrolling was coincidence; the trigger was the promise not landing.

The bet is invisible wherever fetches reliably succeed, which is exactly the
developer's machine. This one failed only off the development platform, and
the first outside tester hit it in her first five minutes.

## The Pattern

1. **Build the replacement off-screen; swap it in whole.** Assemble into a
   detached fragment and install it in one operation, so a slow or failed
   fetch never leaves the member looking at an empty pane in the meantime.
   This is `atomic-state-writes` moved from disk to the DOM: temp file plus
   rename, fragment plus replace.
2. **An empty read against a non-empty view means keep the view.** Silence is
   the honest response to a failed fetch, not erasure.
3. **A live producer owns the surface.** If something is actively streaming
   into the view, a background refresh doesn't get to touch it at all.
4. **Preserve the member's place.** Only ride to the bottom if that's where
   they already were.

## The Edge Rule 2 Doesn't Cover

An empty read is the easy case, because zero is unmistakable. A *partial* read
is the same failure wearing a plausible face: three turns come back for a
forty-turn thread, the guard sees a non-empty result, and the swap eats
thirty-seven turns. Nothing looks broken, which is worse than a blank pane
because a blank pane gets reported.

The strong form of the rule is that a view is replaced by at least as much as
it holds, or not at all. Guarding only emptiness is the cheap version, and
worth naming as cheap rather than shipping it as the rule. Where the read can
truncate rather than fail outright (a rotated log, a paginated API, a parser
that stops at the first bad record), compare counts, not just zero. See
`a-partial-read-proves-presence-not-absence` — a short read is evidence of
presence and never of absence, and a refresh that trusts one is spending the
member's state on that mistake.

## When to Use

- Any view holding state the member produced or accumulated, refreshed from a
  source that can fail: chat history, a draft, an editing surface, a long list
  someone has scrolled into.
- Anywhere an event handler clears a surface before an async call that fills
  it. The clear and the fill in two statements is the shape to grep for.

## When NOT to Use

- Cheap idempotent surfaces with no member-side state (a clock, a status dot).
  Rebuilding those from nothing costs nobody anything.
- A view whose whole job is to show the fetch result and which has no prior
  content worth keeping. Then an empty result *is* the answer, and holding the
  last one would be the lie.

## Adjacent Patterns

- `atomic-state-writes` — the same swap-in-whole discipline on disk; a view is
  state too, and a half-written one is a corrupt one.
- `a-partial-read-proves-presence-not-absence` — why the emptiness guard is a
  floor and not the rule.
- `a-derived-location-is-a-guess-not-an-address` — what actually made the
  fetch fail here, and the half of this incident that generalizes furthest.
- `phase-is-computed-ui-is-rendered` — the sibling failure where the surface
  lies about state instead of losing it.
- `state-published-as-an-event-is-lost-to-latecomers` — the other way a view
  ends up showing less than the truth.

## Source

The dashboard PR #6, merged 2026-08-19: a finished turn cleared the
pane synchronously and then read history, so a failed read replaced a live
conversation with the one-line fallback. The fix builds into a detached
fragment, returns early when the read is empty against a non-empty pane,
yields the surface to an active stream, and only scrolls if the member was
already at the bottom. First outside contribution to the repo, shipped by the
tester's own agent the same day the tester started.
