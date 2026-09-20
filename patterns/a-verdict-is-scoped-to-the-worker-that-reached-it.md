---
type: pattern
date: "2026-08-05"
source: The personal agent's two-host transcription pipeline — a shared terminal-state marker read across hosts silently inherited a peer's give-up state, found while dogfooding a two-machine setup, 2026-08-05
tags:
  - architecture
  - reliability
  - anti-pattern
  - multi-agent
  - coordination
---

# A Verdict Is Scoped to the Worker That Reached It

Two hosts transcribed from one shared, synced folder using different engines — that difference was the entire reason the second host existed, since it handled long audio the first couldn't. A "terminal, stop retrying" state was introduced so a file that had failed enough times would stop burning cycles. It was stored per-host. It was also read across hosts: whenever a host had no marker of its own for a file, it read whichever peer's marker existed and treated that peer's verdict as its own.

The laptop, on its first attempt at a long file, read the leader's `attempts=1089`, cleared its own 15-attempt ceiling in one read, and gave up on audio its own engine had never actually tried.

## The Pattern

**One worker's "give up" must not bind a differently-capable worker, and one worker must not delete another's history.**

- A terminal verdict is a fact about *this worker's* attempts with *this worker's* capabilities. It says nothing about whether a worker with a different engine, budget, or model would also fail.
- Reading a peer's marker "because I don't have my own yet" treats absence as inheritance. It isn't — absence means this worker hasn't tried, not this worker has tried and failed 1,089 times.
- Two hosts sharing one legacy marker path compounds it. Deleting your own terminal marker to reset a file syncs as a delete to your peer too, so the peer's independent attempt count silently resets to zero, erasing the exact number that made a stuck item visible.

This generalizes well past hosts. **A cheap model's "I can't do this" must not bind the expensive model called in afterward. One shard's failure budget is not another shard's.** Cross-worker aggregation is a reporting concern — "terminal on both" is a genuinely useful view — and belongs in the reporting layer, never inside a single worker's own skip decision. And when a report does aggregate attempt counts across workers, take the max, not the sum: a retry count is a floor on how many times *that worker* has tried, and two floors from two independent workers added together aren't a floor on anything.

## The Green Check That Certified the Bug

A 60-second stability check passed, repeatedly, on the broken version. It passed because the laptop was sitting inside a backoff window it had read straight off the leader's marker — the exact condition the peer-read bug produces looks, from a test's point of view, identical to correct behavior. The per-host assertion the suite already had could only ever run with no peer marker on disk, so its negative case, the one that would have caught this, could not occur in that test's setup.

The cure isn't a better assertion, it's proving the existing one is load-bearing: delete the line it guards and confirm the suite goes red. If it stays green, the assertion was checking something that couldn't fail, which is exactly what happened here.

## Related

- [[coordinate-agents-through-shared-state]] — the healthy version of two workers sharing state: append-only, attributed by side, disagreement preserved rather than resolved. This pattern is what happens when a shared surface instead lets one side silently overwrite or inherit the other's
- [[symmetric-reference-gets-used-backwards]] — a different shape of the same root cause: a fact recorded without which-side-you're-standing-on gets misapplied at the point of use. Here the misapplied fact is a peer's own terminal state
- [[a-second-writer-satisfies-your-gate]] — same family: a store with more than one writer satisfies a check that assumed one, silently

## Source

The personal agent's two-host transcription pipeline: a laptop and a "leader" host, each running a different speech engine against one synced audio folder, the second engine existing specifically to handle long files the first couldn't. The terminal-state marker was per-host in storage but cross-host in read path, so the laptop's first attempt at a long file inherited the leader's `attempts=1089` and gave up under its own 15-attempt ceiling without trying. A separate shared legacy marker path meant either host deleting its own terminal marker synced as a delete to the peer, silently zeroing the peer's independent attempt count. A 60-second stability check passed throughout because it only ever ran with no peer marker present, so its negative case never fired, confirmed by deleting the assertion and finding the suite stayed green. Found while dogfooding the two-machine setup, 2026-08-05.
