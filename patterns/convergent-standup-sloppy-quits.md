---
type: pattern
date: "2026-08-01"
source: The personal agent — walk-and-talk: a phone-driven quit half-happened, the next session's voice went to yesterday's brain; fixed by making start converge instead of demanding clean stops
tags:
  - lifecycle
  - resilience
  - agents
  - multi-session
---

# Convergent Stand-Up, Sloppy Quits

A system whose correctness depends on the *previous* run having ended cleanly
has put its invariant in the one place nobody reliably executes. Quits are
where attention has already left: the app gets closed, the phone gets pocketed,
the terminal gets killed mid-teardown. Designing a better stop ritual improves
the median and does nothing for the tail — and the tail is every real incident.

Put the intelligence in **start**. Start is where attention arrives.

## The Problem

A walk-and-talk stack (tmux session + audio bridge + resident synth) was
stopped from a phone. The stop half-happened: the bridge died, the tmux twin
and synth survived. The next session found the bridge "already up" — wired to
inject voice into the *previous* session's brain. The user's words would have
gone to yesterday's conversation, silently, while the new session waited for
input that never came.

The instinctive fix — "make stop more thorough" — cannot win. The stop that
half-ran wasn't buggy; it was *interrupted*, which no amount of stop-side code
survives.

## The Pattern

Stand-up owns four jobs, all deterministic:

1. **Idempotent births.** Every component's start script checks for a live
   instance first (`already up` is success, not conflict), so converging costs
   nothing when the world is already right.
2. **Liveness-checked wiring, newest-intent-wins.** Don't just check that a
   dependency *exists* — check that what it points at is still alive. A bridge
   whose injection target is a dead session gets replaced; one pointed at an
   *abandoned but live* session gets rewired to the newest stand-up, **with an
   announcement** naming what was displaced. The user who just started
   something is the user the resources belong to.
3. **Inventory the leftovers, classified by measured recency.** Not "sessions
   exist" but *this one, 12 minutes idle, holds the ears; that one, 3 hours
   idle, is stale*. Classification uses the platform's own activity clock
   (tmux `session_activity`), never name-guessing. Emit **one computed
   recommendation** — resume X / start fresh / name the keeper — that the
   agent relays in a sentence and the human overrides in a word.
4. **Reap the stale, announced.** Leftovers beyond a threshold are dropped at
   stand-up with a line each ("dropped 'walk' — idle 3h"). Silent cleanup
   erodes trust exactly as much as silent leaking; the announcement is what
   makes autonomous cleanup acceptable.

Expensive resources get a fifth rule: **linger, bounded.** A 46-second model
load re-paid on every restart is the tax users feel most. Retire it on a
guarded timer — cancelled by the next start, refusing to fire while anything
live depends on it — so the worst case is one bounded window, not an orphan
and not a cold start per restart.

## The Test

Kill the stack at any point — mid-stop, mid-start, `kill -9`, close the
laptop — then run start once. It must end in the same working state every
time, and everything it decided (rewired, dropped, resumed) must appear in its
output. If any sequence of dirty exits can make start produce a different
state, the convergence has a hole exactly there.

## Related

- [[teardown-verified-startup-never-written]] — the lifecycle wired at only one end
- [[announce-the-move-in-the-old-room]] — relocations that strand a UI
- [[health-check-that-never-exercises]] — "exists" mistaken for "works"
- [[declared-presence-beats-host-clock]] — measured activity over inference

## The Rule of Thumb

If your runbook says "make sure to shut down cleanly," the design owes you a
start that doesn't care. Clean shutdown is a courtesy; convergent start is the
contract.
