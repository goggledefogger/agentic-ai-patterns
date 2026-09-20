---
type: pattern
date: "2026-08-04"
source: The personal agent's voice-capture pipeline (voice_autotriage.py, a doc, spec-capture-integrity CAP-6)
tags:
  - architecture
  - routing
  - local-models
  - safety
---

# A Model Never Picks the Destination

Any capture pipeline — voice notes, photos, links, meeting recordings — reduces to four verbs: **capture, transcribe, understand, route**. The load-bearing decision is which verbs get a model at all, because that decision is also the offline story, the privacy story, and the safety story, all at once.

## The Pattern

Split the verbs by what they actually need:

| Verb | Needs a model? | Runs where |
|---|---|---|
| Capture | No — transport (sync, bot, watcher) | Device / P2P |
| Transcribe | Yes — completion-shaped (audio in, text out) | **Local, always** — the content hasn't been classified yet, so it must be treated as maximally sensitive; it cannot ride a cloud API before anything knows what it is |
| Understand (summarize, extract concepts) | Yes — completion-shaped | Local by default, cloud by explicit choice per task |
| Route (pick the destination, the sensitivity tier, the approval gate) | **No — and never** | Deterministic lookup against a registry |

The routing verb is the one people reach for a model to do, and it is the one verb a model must never do. A destination is also a *sensitivity decision* — file this to the shared repo vs. the private vault — and a guessed tier is a leak with extra steps. A registry lookup is auditable, works offline by construction, and cannot hallucinate a new home. The model's only role near routing is producing the **summary the lookup reads** — it proposes nothing.

Two corollaries earned the hard way:

- **Summaries must be rich enough to carry the routing.** A one-line gist routes the *handle* ("the family member recording" → the person's vault); a structured summary (participants, purpose, topics, decisions) routes the *content* (it was course material). Underspending on the understand verb silently corrupts the route verb.
- **Measure the local share honestly, or "local by default" is a vibe.** A daily grader that counts every model call (local vs. cloud, per project) turns the doctrine into a number — and the first honest number is usually an F. That's a baseline, not a failure; per-task migration guided by grades beats a big-bang "go local" that quietly degrades quality.

## Offline Follows for Free

Because capture is transport, transcribe is local, and route is a lookup, the entire unattended path runs with zero cloud: a note spoken during an outage still lands transcribed with its destination proposed. The only thing an outage removes is the conversational layer on top — which is the correct thing to lose.
