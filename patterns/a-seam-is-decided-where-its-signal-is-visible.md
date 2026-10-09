---
type: pattern
date: "2026-10-09"
source: A chat dashboard streaming text from coding agents; words glued together across tool calls, fixed twice, once per protocol
tags:
  - code
  - debugging
  - protocols
  - streaming
---

# A Seam Is Decided Where Its Signal Is Visible

Streamed text from coding agents glued words together across tool calls: "...complete.Checking...". The agent had said one thing, run a tool, then said another, and the display joined them with nothing between.

Two lanes fed the same display, and they needed different fixes.

**Lane one:** the native command-line stream, which sends explicit block-start events. The browser could see that a text block had opened after a thinking block, so it started a new paragraph there. 

**Lane two:** an adapter speaking a different protocol, where several agents arrive through one bridge. It delivers all its text as one block, with no block-start events. The browser fix could not fire, because the seam was not in the data it received. The fix had to live in the bridge, where the protocol's own events were still visible: a tool completed, a thought chunk arrived, the message id changed.

## Why the wrong moves were attractive

**One shared join rule, downstream.** Tidy, but it had nothing to act on: the adapter had already flattened the signal before the shared code saw it.

**Recover the signal from timing.** A pause of 400 ms followed by a capital letter means a new paragraph. This was proposed and rejected. It fails on slow streams, where a pause is only slowness, and it splits "I" in the middle of a thought. It also reaches for a guess when the protocol declares the answer, the same trap as [a heuristic where an exact key exists](a-heuristic-where-an-exact-key-exists.md).

## The rule

**Decide a seam where the signal that defines it is still visible.** One rule per protocol boundary is correct when the boundaries carry different signals. Consolidation is a fine goal only after you check that the signal survives to the place you consolidate.

**Never reconstruct a flattened signal from timing.** If the structure was lost, restore the loss at its source, or say plainly that the seam cannot be known.

## The open question

The bridge uses a changed message id as one cue, since the protocol says a changed id indicates a new message. Whether real agents send it is unmeasured, so the question is filed, not assumed.

## The tell

A downstream fix that "works on the fixture" and does nothing on the second lane. Ask of any join, split, or merge rule: what event tells it where to act, and does that event reach this layer?

## Adjacent Patterns

- [A Heuristic Where an Exact Key Exists](a-heuristic-where-an-exact-key-exists.md)
- [A Partial Read Proves Presence, Never Absence](a-partial-read-proves-presence-not-absence.md)
