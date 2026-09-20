---
type: pattern
date: "2026-08-05"
source: The personal agent's walk-and-talk watchdog (walk-watchdog.sh, voice-log.jsonl, spec-walk-silent-turn-incident)
tags:
  - observability
  - watchdogs
  - voice
---

# Liveness Is Measured at the Ear

A watchdog that measures the producer's activity certifies the producer, not the experience. A session doing heavy agentic work — tool calls, edits, a growing transcript — looks maximally alive by every producer-side signal while the human consumer receives nothing at all. The healthier the work, the more convincingly the watchdog lies.

## The Pattern

For any pipeline whose output is consumed live (voice replies, streamed UI, notifications), the watchdog needs a signal measured at the **consumer's end of the pipe**: time since the last thing the human actually received — the last spoken utterance, the last rendered frame, the last delivered message. Producer-side signals (transcript growth, CPU, log lines) stay as a *second* check; they catch a dead producer, which is a different failure than a silent one.

Two signals, two failure modes:

| Signal | Catches | Blind to |
|---|---|---|
| Producer activity (transcript mtime, log growth) | Wedged / crashed producer | Healthy producer that stopped delivering |
| Consumer delivery (time since last utterance reached the device) | Silent-but-busy turns | Nothing — but needs the delivery log to exist |

The corollary that made this incident diagnosable at all: **log delivery, not just production.** The fix was only possible because the voice log recorded what reached the phone, separately from what the session produced.

## Where It Was Earned

2026-08-05: The personal agent's walk watchdog checked injected-input age against transcript growth. A session spent four minutes on tool-heavy work — transcript growing continuously, zero speech delivered — and the watchdog, by construction, saw perfect health while the walker gave up and closed the page.
