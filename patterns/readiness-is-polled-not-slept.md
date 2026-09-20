---
type: pattern
date: "2026-08-07"
source: The personal agent — walk-and-talk phone bridge and day-review cold start, both waited on a flat sleep instead of the service's own readiness signal, found 2026-08-07 (spec-walk-readiness-polling)
tags:
  - lifecycle
  - verification
  - services
  - anti-pattern
---

# Readiness Is Polled, Not Slept At

After launching a service, wait on the service's **own** readiness signal in a bounded loop that fails loudly. Never a flat `sleep(N)`.

A flat sleep is wrong in both directions at once. When the service comes up faster than the guess, you waste the difference on every single launch. When it comes up slower — or doesn't come up at all — you sail past the sleep and proceed against a dead service, and nothing tells you that's what happened.

## The Problem

Measured 2026-08-07, walk-and-talk skill: roughly 20 seconds of the walk's two longest silent waits were hardcoded sleeps, on two services with nothing in common except that someone had to guess a number for each.

- **Phone audio bridge.** Launched, then a flat `sleep 12`. The bridge actually converges in 2-4 seconds and writes `phone-url.txt` the moment it can serve. The other 8-10 seconds were pure waste, paid on every walk.
- **Day-review server.** Cold-started with a flat `sleep 8`. It boots in 1-2 seconds and answers HTTP as soon as it's up. Same shape, same waste.

Neither number was wrong by accident. The root cause sits one level up: the docs for both said "launch it" or "ensure it's up," with no recipe for *how to know* it's up. Each assistant session that touched the code filled that gap with a guess-sized sleep, because a guess is the only thing "ensure it's up" leaves you to write.

## The Pattern

**Bounded poll on the service's own signal, not a timer.** Roughly 0.5s steps, roughly a 10s cap, checking the thing the service produces when it's actually ready — a file it writes, an HTTP 200, whatever its real contract is. When the cap is hit, fail loudly: surface that the service never came up, don't fall through as if it had.

**Never a process check.** A live pid proves something started. It doesn't prove it can answer — see [[health-check-that-never-exercises]] for the same gap one layer up, where the process is up but the work it exists to do isn't.

**Corollary 1 — clear the readiness signal before launching.** A file-based signal left over from the previous run reads as instantly-ready to a poll that only checks for existence, and you'll ship exactly that. This repo did, 2026-08-03: a stale `phone-url.txt` from a prior session got hand-delivered to a fresh one, and it only worked because the token inside happened to still be valid. That's [[live-reference-outlives-its-owner]] wearing a readiness-file costume — the reference outlived the process that wrote it, and nothing at read time checked whether it still pointed at something live.

**Corollary 2 — fetch once, share the result.** Once the poll confirms readiness, pull the data into a file or variable once and hand it to every consumer. Polling-then-refetching per consumer is the same waste as the flat sleep, just moved: you already paid to know it's ready, don't pay again per reader.

## Related

- [[health-check-that-never-exercises]] — a process that's up isn't a process that works; a pid is not a readiness signal
- [[teardown-verified-startup-never-written]] — the birth/death symmetry gap this pattern's failure mode sits inside: nobody wrote the recipe for "up," so nobody could poll for it
- [[live-reference-outlives-its-owner]] — the stale `phone-url.txt` is this pattern's readiness-file cousin: a leftover signal read as current

## Source

The personal agent a personal skills repo walk-and-talk skill (phone bridge launch) and the personal agent day-review skill (cold-start server launch), both replacing a flat `sleep` with a bounded poll on the service's real readiness signal. `spec-walk-readiness-polling`, 2026-08-07.
