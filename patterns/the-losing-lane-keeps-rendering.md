---
type: pattern
date: "2026-08-05"
source: The household-agent repo Hub — a streaming download finished an episode while the abandoned torrent for the same episode kept showing as a stopped multi-GB download for 4m22s
tags:
  - code
  - ux
  - anti-pattern
  - operations
  - monitoring
---

# The Losing Lane Keeps Rendering

A system with a primary path and a fallback eventually runs both at once. One
wins. The question nobody designs an answer for is what happens to the
**loser's state object** — and the usual answer is "a periodic sweep gets to it
later." Until it does, the interface shows the abandoned attempt, which is
almost always the uglier of the two: stalled, stopped, enormous, with a retry
button. The user reads the failure, not the success sitting quietly beside it.

## The Problem

Two lanes could deliver one TV episode: a streaming scraper and a torrent. Both
were started for a single request. The torrent was hopeless — 0 seeds, a
hundred-day ETA — and the stream delivered in six minutes.

The active-downloads panel skipped torrents only at `progress >= 1.0`. This one
was a *selective grab of a single file out of a season pack*, so it would never
reach 1.0 no matter what happened. It kept rendering: the pack's name, several
gigabytes, state `stopped`, and — because paused torrents are resumable — a
**Resume** button offering to restart a download the system had deliberately
abandoned.

The row finally vanished four minutes and twenty-two seconds after the episode
was complete and playable, when a five-minute cron swept it. The whole time,
the correct answer existed in a different subsystem and the display had no way
to hear about it.

Three separable defects, and it is worth keeping them apart:

- **Nobody decided both lanes could run.** A skill document offered two recipes
  for the same task and never said "pick one," so an agent ran both. The
  fallback lane was already wired to trigger itself when the primary exhausted;
  firing it manually defeated its own guard.
- **The winner's completion was not an event the loser could observe.** Each
  lane owned its own state and neither published "this entity is done" anywhere
  the other read.
- **Retirement was owned by a poll.** The sweep interval silently defines the
  maximum duration of the lie, and nobody had ever written that number down as
  a cost.

## The Pattern

**Completion in any lane must retire the others' state at completion time, not
at the next sweep. Where you cannot safely delete, suppress the display.**

1. **Cross-lane completion needs a shared key.** Each lane wrote its own
   identifier format; the fix was making both spell the entity the same way
   (`<Series> - SxxEyy`) in one shared, append-only ledger. Now one `grep` shows
   every lane that touched an entity, and the display layer can ask "has anyone
   finished this?"
2. **Suppress narrowly, on evidence of both facts.** Hide a row only when it is
   *not progressing* **and** every entity it covers is already delivered. A lane
   still moving bytes is never hidden — otherwise a legitimate retry disappears
   because an older copy exists.
3. **Hiding is not deleting, and sometimes that is the whole point.** A reaper
   already existed for "another path delivered this" — and deliberately exempted
   pack torrents, because deleting a pack destroys the seed source for every
   other file previously taken from it. The tempting fix ("extend the reaper")
   is the dangerous one. **Display-only suppression is the correct half when
   deletion is unsafe**, and that reasoning belongs in the code, or the next
   reader will "finish the job."
4. **Log the suppression.** A row vanishing from a panel must be explainable
   from a log afterwards, or you have traded a confusing display for an
   unfalsifiable one.

## When It Shows Up

Any primary/fallback pair that can overlap: a CDN fetch racing a origin fetch,
a cache warm racing a live query, two payment processors, a retry that
succeeds while the original attempt is still queued, mirrored uploads, a job
retried on a second worker. Also any UI whose "active" filter is defined by a
*progress threshold* rather than by a terminal state — partial-selection cases
never reach the threshold and become immortal.

The tell: ask "when lane A wins, what deletes lane B's row, and how long does
that take?" If the answer is a cron interval, that interval is the maximum
duration of a visible falsehood, and you should be able to say the number out
loud.

## Related

- [[disproving-the-theory-is-not-disproving-the-sighting]] — how this was
  nearly filed as a non-event
- [[external-store-as-source-of-truth]] — one place both lanes can publish to
- [[stale-pointer-asserts-confidently]] — stale state that presents as current
- [[unknown-value-renders-as-absence]] — the near-inverse: a valid record that
  renders as nothing, where this one is a dead record that renders as live
- [[replica-freshness-travels-with-the-count]] — carrying staleness with the data
- [[one-authority-for-repeating-behaviors]] — one lane per task, decided once

## The Rule of Thumb

Whoever wins the race owes the losers a retirement. If that debt is paid by a
poll, the poll interval is how long your interface is allowed to lie.
