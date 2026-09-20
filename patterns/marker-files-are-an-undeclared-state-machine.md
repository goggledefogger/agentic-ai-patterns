---
type: pattern
date: "2026-08-03"
source: The personal agent's walk-and-talk skill (ux-debt 70 — the double "welcome back")
tags:
  - code
  - reliability
  - state
  - multi-agent
---

# Marker Files Are an Undeclared State Machine

A system that coordinates through marker files — `loop-confirmed`, `handshake-spoken`, `link-pushed`, `owner-session` — has a state machine whether or not anyone declared it. Each marker is a state; each script that writes one is a transition owner. Leave it undeclared and the transitions interleave: five writers, no arbiter, and any two processes can each believe they own the walk.

## The failure that named it

Walk-and-talk's stand-up creates a twin — a second Claude process resuming the same transcript so a phone can attach. The night of 2026-08-03, both brains walked the same stand-up checklist, because the checklist lives in the shared transcript and nothing recorded that it had already been walked:

- The original stood everything up and spoke the greeting. The walker answered.
- The twin, arriving two minutes later, ran `session.sh start` — whose stray-sweep asked a **teardown** auditor ("is everything down?") a **lifecycle** question ("is anything wrong?"). The healthy live bridge read as FAIL, so start killed it mid-conversation, rebooted it, and spoke the identical greeting again. The walker's answer had died with bridge #1; he was asked for it a second time, verbatim.
- At teardown, the record that would have let `stop` end the other brain (`twinned-from`) had been overwritten by the twin's own pass through the same script — the record destroyed by the thing that creates it.

Three defects, one root: states existed, ownership didn't.

## The Pattern

Declare the machine. Three rules cover most of it:

1. **Every marker names its owner and epoch.** An epoch bumps each time the underlying resource is (re)born — a bridge boot, a server restart. `handshake-spoken@epoch-3` grants nothing in epoch 4. This kills the whole class of stale-readiness bugs: a marker can no longer outlive the thing it described.
2. **Arriving at an owned state converges — it never re-runs the transition.** A second process finding the walk live *adopts* it (or stays silent). Stand-up is a transition, not a script anyone may re-execute; running it from a state that is already past it is a no-op, not a restart.
3. **Records that identify the owner are write-once while the owner lives.** Overwrite only after verifying the recorded owner is actually gone — by identity (pid *and* start time), never pid alone.

And the corollary that caused the kill shot: **a verifier's expectations are relative to a state.** "Everything down" is correct in IDLE and catastrophic in WALKING. A check that doesn't know which state it is verifying will report the healthy system as wreckage — and a caller that trusts it will finish the job.

## When it applies

Any time coordination happens through the filesystem: lock files, `.ready` markers, pid files, "confirmed" flags — especially when more than one process (or more than one *agent*) can run the scripts that write them. Multi-agent setups sharing a transcript or a repo hit this hard: the instructions that led agent A through a sequence are visible to agent B, and B will helpfully re-run them unless the state says the sequence is owned and done.
