---
type: pattern
date: "2026-08-07"
source: The personal agent walk-and-talk voice assistant, live walk — a spoken "give me a few seconds" landed 26s after the question, delivered as a turn-closing reply, ahead of a 3m17s silent wait with the watchdog disarmed
tags:
  - voice
  - latency
  - agents
  - liveness
---

# Ack Then Work

The acknowledgment is the cheapest thing a consumer-facing agent will produce all turn, and the only thing it can produce before the slow work is even dispatched. A commitment spoken after the long turn has already started isn't a commitment, it's a status update — and the consumer's clock started at their last word, not at the agent's first tool call.

## The Problem

A walker asked a question. The session dispatched a subagent, then composed and delivered the spoken ack — "give me a few seconds" — 26 seconds *after* the question, and delivered it as a normal reply. The page correctly treats a normal reply as end-of-turn, so speaking the ack disarmed the still-working watchdog at the exact moment it was needed. The session then waited 3m17s on the subagent in total silence, watchdog off. The walker got 26 seconds of dead air, a promise, and then over three minutes of nothing, with no safety net covering it.

Nothing here was a slow model or a slow subagent. Both were doing what they were dispatched to do. The failure was entirely in when the ack went out and what it did when it arrived.

## The Pattern

Two requirements, both necessary, neither sufficient alone:

**Ordering — the ack goes out before the dispatch.** It's the cheapest thing the agent produces all turn, so there's no reason to hold it behind anything else. Emit it, *then* dispatch the subagent. Voice-latency work agrees on the mechanism: perceived speed is measured as time-to-first-audio, not time-to-final-answer, and a filler acknowledgment makes an identical wait feel roughly half as long. The ack is not politeness bolted onto the front of the turn, it's the thing that makes the wait survivable.

**Marking — the ack must carry a "turn still open" flag.** A spoken ack delivered through the normal reply path *is* a normal reply as far as downstream liveness machinery is concerned, and a normal reply ends the turn. If the ack closes the turn, it switches off the exact watchdog that would have covered the silence it just announced — the fix and the failure are the same channel. Mark it (a `--working` flag or equivalent) so the turn stays open and the liveness signal keeps covering the wait after the words stop.

Skip either half and the ack does nothing: skip the ordering and the walker is already annoyed before the promise arrives; skip the marking and the promise itself is what disarms the net.

## Related

- [[liveness-is-measured-at-the-ear]] — why the consumer's clock, not the producer's, is the one that decides whether the wait was silent
- [[self-report-must-not-end-what-it-reports-on]] — the ack closing the turn is that pattern's failure wearing a politeness costume: a self-report running the same teardown a real completion would

## Source

The personal agent walk-and-talk live observation, 2026-08-07 23:08–23:15 (spec-walk-working-ack-binding, patterns-consult gap analysis the same night).
