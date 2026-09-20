---
type: pattern
date: "2026-07-27"
source: The personal agent — voice-capture inbox read off a stalled Syncthing replica, 2026-07-26/27
tags:
  - replication
  - drift
  - monitoring
  - verification
  - anti-pattern
---

# A Count Inherits the Freshness of the Replica It Was Counted From

When the data you read is a replica, every answer you compute carries a hidden qualifier: *as of the last time this replica synced*. Nothing in the answer's shape shows that qualifier. A stalled replica does not return an error, a partial result, or a warning. It returns a **complete, well-formed, confidently wrong** answer that is indistinguishable from a correct one.

The dangerous case is the low count, because a stalled replica and a quiet system produce the same number, and the same relief.

## The Problem

The personal agent triages a voice-note inbox replicated between a phone, a cloud leader, and a laptop over Syncthing. On 2026-07-26 the sync daemon was not running on the laptop. The tool reported `4 note(s) pending triage`. That was a true statement about the files on disk and a false statement about the world: the real answer was 5. The fifth had been sitting on the leader, and it appeared the moment sync was restarted.

Nothing about the wrong answer looked wrong. Four is a plausible number of notes. The list rendered cleanly. Had the session triaged what it saw and reported "inbox clear," the missing note would have been skipped — and skipped *silently*, because triage stamps notes as handled, and a note that was never seen is never stamped, so it does not resurface as an anomaly. It just stays behind.

There was a second trap layered on it. The obvious diagnostic was wrong. The documentation asserted that this replication edge rode a VPN, so the natural check was VPN status — which was indeed down, which *confirmed the wrong theory*. Measurement showed the sync daemon had been reaching the leader over its public address all along. The VPN mattered for an unrelated health probe. Roughly five minutes went into the confidently-documented wrong lever before anyone measured the actual one.

## The Pattern

**Whatever surface reports the count reports its own trustworthiness, in the same breath.**

```
inbox: FRESH (in sync with 1 peer(s), 0 files pending)
2 note(s) pending triage
```

```
inbox: STALE (sync daemon not running — this copy is FROZEN)  <-- do not trust the count below
2 note(s) pending triage
```

Four properties carry the weight:

**1. Attached, not adjacent.** Not a second command someone must remember, not a separate dashboard. The freshness verdict is emitted by the same call that emits the count, because the moment of reading is the only moment the qualifier matters. A freshness check nobody runs is worth exactly nothing, and "remember to check first" is a rule that survives about a week.

**2. Four states, not two.** `FRESH` / `SYNCING` / `STALE` / `UNKNOWN`. Only `FRESH` licenses acting on the number. `SYNCING` means the count is a moving target. `UNKNOWN` is the one people skip and the one that matters most — see below.

**3. A sub-check that fails is UNKNOWN, never a finding.** The first implementation asked two questions: is the folder synced, and is a peer connected? When the peer lookup threw, the code counted zero peers and reported `STALE — no peer connected`. That is a fabricated diagnosis: a lookup that never completed, laundered into a confident claim about the world, sending the next reader after a peering problem nobody observed. A check that did not run must never borrow the verdict of a check that ran and found nothing.

**4. Advisory, never a gate.** The verdict does not block the read. A staleness check that can wedge the pipeline becomes a liability the first time its own dependency is down, and someone will disable it — taking the honest signal with it.

## Do not diagnose replication from the transport you assume it uses

The corollary the VPN detour teaches: **measure which edge is actually carrying the data before believing any document about it, including your own.** Replication layers negotiate their own paths — direct, relayed, hole-punched, public, private — and a written claim about *how* a replica syncs decays silently, because nothing breaks when it stops being true. The documentation had been wrong for weeks and was believed because it was specific.

The check that pays for itself is the one that asks the replicator *what it is currently doing*, not the one that asks whether the transport you believe it uses is up.

## Verification

Exercise the stale path against the real thing. Stop the sync daemon, read the count, confirm the warning appears, restart, confirm it clears. A staleness check tested only by fixtures asserts that a mocked failure produces a warning — never that the check can perceive the failure it exists for. The personal agent's was verified this way, and only then was the four-vs-five discrepancy explicable rather than a mystery.

Include the negative control: a healthy replica must produce **no** warning. A verdict line that appears unconditionally is one readers stop seeing within a day.

## Adjacent Patterns

- **Count the source, not the survivors** (`count-the-source-not-the-survivors.md`) — the other way a count is honest about the wrong population. Both were found in one session, from the same inbox, and neither would have caught the other's case.
- **Verification needs a negative control** (`verification-needs-a-negative-control.md`) — why the FRESH-is-silent half is load-bearing.
- **A stale pointer asserts confidently** (`stale-pointer-asserts-confidently.md`) — the same failure in prose rather than data: a copy that keeps answering after it stopped being right.
- **Self-reporting staleness check** (`self-reporting-staleness-check.md`) — the scheduled-sweep form; this is the read-time form, and the two want different triggers.
