---
type: pattern
date: "2026-08-07"
source: The personal agent walk-and-talk stand-up — one turn ran `session.sh start` twice (both returning "already LIVE — adopting"), re-ran a voice-loop confirmation that had already passed, and re-probed bridge status it had just read, ~15 of ~54 total seconds spent re-proving state proven moments earlier
tags:
  - efficiency
  - agents
  - state
  - voice
---

# Session-Scoped Result Cache

A fact an agent has already verified this session — a service is up, a loop is confirmed, a config value is set — is held and reused, not re-derived by running the same check again. Re-verification costs a full tool round-trip per fact, and between checks separated by seconds, nothing changed.

## The Problem

A walk-and-talk stand-up ran `session.sh start` twice in one turn, both calls returning "already LIVE — adopting." It re-ran the voice-loop confirmation after that same confirmation had already passed, and re-probed bridge status it had just read two lines earlier. None of these were wrong answers — every re-check returned the same thing the first check had. They were just spent again: roughly 15 of the turn's ~54 seconds went to re-proving state that was already proven, each re-check costing a model round-trip plus a subprocess for zero new information.

## The Pattern

Hold the answer once a check has actually run, and reuse it for the rest of the turn instead of re-invoking the check. The scoping is the hard part and where the pattern earns its name:

- **The cache is session-scoped, not durable.** It lives for the length of this turn/session and nothing longer — a fact proven a minute ago in *this* session is trustworthy; the same fact carried into a new session is a stale-marker problem instead, [[verify-freshness-before-acting]] territory, not this pattern.
- **The agent's own actions invalidate it.** Restarting the bridge invalidates "bridge is up." A cache that doesn't fall the instant the agent itself changes the underlying state is worse than no cache — it's a lie the agent told itself.
- **Explicit external signals invalidate it too.** If something outside the agent's own actions plausibly changed the fact (a user says "I restarted it," a watchdog fires), re-check for real. The cache covers the ordinary case — nothing happened between two checks seconds apart — not every case.

Distinguish this from **[[readiness-is-polled-not-slept]]**, which covers the *first* proof: poll a service's own signal until it's actually ready, rather than sleeping a guessed duration. That pattern is about earning the first "yes." This one is about every "yes" after it — once the first proof lands, stop re-earning it. The fetch-once corollary in that pattern (don't re-fetch data you already pulled this turn) is this same discipline applied to data instead of checks.

## Related

- [[convergent-standup-sloppy-quits]] — the idempotent-birth half of this incident: `start` returning "already LIVE" once is correct and cheap; the bug here was asking it a second time in the same turn
- [[readiness-is-polled-not-slept]] — covers proving the fact the first time; this pattern covers not re-proving it after
- [[verify-freshness-before-acting]] — the cross-session sibling: what to do when the held fact might be stale because time, not just the turn, has passed

## Source

The personal agent walk-and-talk live observation, 2026-08-07 23:08–23:15 (spec-walk-working-ack-binding, patterns-consult gap analysis the same night).
