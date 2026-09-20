---
type: pattern
date: "2026-08-07"
source: The personal agent — walk-and-talk: a pending permission dialog was broadcast once on change, so a phone page that reloaded into an already-frozen session saw nothing and opened its microphone in front of it
tags:
  - distributed
  - resilience
  - agents
  - interfaces
---

# State Published as an Event Is Lost to Latecomers

An event says *something changed*. State says *something is true*. Publishing
state through an event channel works perfectly for everyone who was listening
at the instant it changed, and is invisible to everyone else — forever, because
there is no second announcement. The bug does not appear in testing, because
the tester is always already connected.

## The Problem

A walk-and-talk bridge surfaces the agent's permission dialogs to the walker's
phone as a tappable panel — a real safety feature, since a dialog freezes the
session and is otherwise invisible from a sidewalk. It was implemented as a
poll that detects a *change* in the pending dialog and broadcasts it,
deliberately unbuffered so control messages would not pollute the replay
history.

The walker's page disconnects constantly on a real walk: screen off, app
switch, reconnect. He reloaded into a session that had been frozen behind an
approval for several minutes, tapped Begin, and got an open microphone. He
talked to a session that could not hear him:

> the first thing it did was listen to me and i had to say something

The dialog had been announced once, to zero subscribers. Nothing was broken.
Nothing logged an error. The state was simply unreachable.

The assistant made it worse by predicting the panel would appear — reasoning
from the feature's existence rather than from its delivery semantics. A
confident forecast of something structurally impossible.

## The Shape

Ask of any published fact: **is this a transition, or a condition?**

- A transition ("the user said X", "a file changed") is genuinely an event.
  Missing it is survivable, and replaying it later would be wrong.
- A condition ("a dialog is pending", "the turn is still open", "the job is
  paused", "we are in degraded mode") is state. A subscriber that missed it is
  not merely behind — it holds an actively false model of the world and will
  act on it.

The tell is the consequence of missing the message. If a latecomer who never
hears it behaves *incorrectly* rather than merely *later*, it was state.

Corollary that bites in practice: **broadcast-to-zero-subscribers is a silent
success.** Every send-side metric looks healthy. This is the same family as
[[ask-who-received-before-you-send-again]] — delivery accounted at the sender
rather than the receiver — but with a nastier shape, because there is no
retry to observe and no duplicate to notice.

## The Fix

Every connection handshake re-asserts current state, after any replay:

```
on_connect(client):
    send(orientation)          # how long you were gone
    send(missed_events)        # the transitions
    if pending_dialog: send(pending_dialog)     # the conditions
    if turn_open:      send(turn_open)
```

Cheap, idempotent, and it collapses "connect" and "reconnect" into one path —
which matters, because reconnect is the path nobody tests.

Two supporting habits:

1. **Test the latecomer, not the observer.** The regression test must
   establish the state *before* the client connects. A test that connects
   first and then triggers the change passes against the broken code — it is
   [[green-tests-can-mirror-the-same-guess]] wearing an ordering disguise.
   Verify it fails with the fix removed.
2. **Never predict a UI affordance from its existence.** Read the live state
   and report what is actually true. A feature that exists and a feature that
   will reach this client are different claims.

## The Generalization

Anywhere a long-lived condition is pushed to clients that come and go — a
maintenance banner, a rate-limit state, a paused pipeline, a "waiting for your
input" flag, a feature flag — the connect handshake is the only place the
truth is guaranteed to be told. Announcements are an optimization on top of
it, never a substitute.

Related: [[ask-who-received-before-you-send-again]] (delivery is not arrival),
[[latched-state-needs-a-reconciler]] (state that can get stuck needs something
that re-derives it),
[[a-twin-inherits-the-conversation-not-the-channels]] (the channel-side sibling
— found the same walk).
