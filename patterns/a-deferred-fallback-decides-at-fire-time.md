---
type: pattern
date: "2026-08-07"
source: The personal agent — walk-and-talk speak.sh, 45-second Telegram grace-watchers armed per spoken line woke after a clean teardown and texted the walk's parting words as bot messages a minute after the call ended, twice (spec-walk-goodbye-before-teardown, commit eb0139c)
tags:
  - lifecycle
  - resilience
  - agents
  - anti-pattern
---

# A Deferred Fallback Decides at Fire Time, Not Arm Time

Any timer, watcher, or scheduled fallback that outlives the thing it serves
must re-check, at the moment it fires and from state on disk, whether that
thing still wants serving. The world it was armed in is not the world it
wakes in.

## The Problem

A voice walk's speak path arms a 45-second Telegram grace-watcher per spoken
line: *text the walker this line if the phone page never reports playing it*.
It's the designed fallback for a bridge that dies mid-walk — a real failure
mode worth covering.

The walk in question ended cleanly. The goodbye played, session stop tore
down the bridge and the page. The watchers armed during the last minute woke
*after* teardown, found — correctly, by their own lights — that no playback
had been logged since their mark, and texted the walk's parting words as bot
messages a minute after the call had already ended. Twice.

Every component did its job. The watcher watched. The absence of a playback
log was real. The composite was wrong because the decision — *no playback yet
means the walker missed it* — was made at arm time and executed at fire time,
and a deliberate teardown happened in between. The watcher had no way to know
the thing it was covering for had been intentionally ended, because nothing
asked it to check.

## The Shape

A deferred action carries two things forward from arm time to fire time: the
work, and the premise that justifies doing the work. Code review, tests, and
normal operation all exercise the work. Almost nothing exercises the premise,
because the premise is implicit — "if we got here, the thing must still need
saving" — and holds in every case except the one where someone ends the thing
on purpose while the timer is still ticking.

Distinguish this from two neighbors it looks like:

- [[one-authority-for-repeating-behaviors]] kills stale *repeating* chains
  with generation tokens — a liveness identity that makes a second live chain
  structurally impossible. This pattern is about a *one-shot* deferred action
  whose premise can expire. The timer itself is legitimate and singular; its
  worldview is what's stale.
- [[live-reference-outlives-its-owner]] is the read-side cousin: a reference
  that outlives its writer, and readers keep traversing it. Here it's an
  *action* outliving its justification — the watcher doesn't just point at
  something stale, it *acts* on the strength of it.

Also related: [[the-losing-lane-keeps-rendering]] — another case where a
component that was correct in isolation kept presenting a picture the rest of
the system had already moved past.

## The Fix

At fire time, before acting, the watcher re-reads the authoritative on-disk
state — here, `state.json`'s `voice_mode`, which a clean stop sets `false`
and a crash leaves `true` — and skips, with a logged skip-event, when the
world says the service was ended on purpose.

```
on_fire(watcher):
    state = read_state()               # not the flag you captured at arm time
    if state.voice_mode == false:
        log_skip(watcher, reason="clean_stop_since_arm")
        return
    if state.unreadable:
        act()                          # fail toward acting; see below
        return
    if not playback_logged_since(watcher.mark):
        act()
```

Two things make this the lazy fix rather than a new subsystem:

1. **No new IPC, no cancellation registry.** The state teardown already
   writes on a clean stop *is* the cancellation signal. Reading it is a
   stat-and-parse, not a design.
2. **Fail-direction is chosen per stakes, not defaulted.** Unreadable state
   fails toward acting — in this domain a spurious goodbye text is cheaper
   than a lost mid-walk line. A different deferred fallback (one where firing
   wrongly is the expensive direction) would fail the other way. The point
   isn't "always act" or "always skip," it's that the choice is made
   deliberately once, at the check, instead of being implicit in whichever
   branch happened to be easiest to write.

## Source

a personal skills repo walk-and-talk `speak.sh` grace-watcher, spec
`spec-walk-goodbye-before-teardown`, commit `eb0139c`, 2026-08-07. Watchers
armed per spoken line to cover a bridge dying mid-walk instead fired after a
clean session end and texted the walk's own parting line back to the walker
as a stray bot message, twice in one walk.
