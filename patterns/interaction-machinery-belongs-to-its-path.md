---
type: pattern
date: "2026-08-22"
source: The personal agent / the voice stack — the web door's self-interrupt evening: three real client-side fixes, none sufficient, while the sibling CLI door had carried the platform's own answer (half_duplex → NO_INTERRUPTION) since July
tags:
  - architecture
  - audio
  - agents
  - adoption
  - anti-pattern
---

# Interaction Machinery Belongs to Its Path

A system with more than one interactive channel ends up with more than one
*path* — some built on an adopted platform (a realtime API, a hosted bridge),
some owned end to end. Each path comes with an owner for its interaction
mechanics: turn-taking, interruption, echo handling, liveness. The failure
this pattern names is **crossing mechanisms between paths** — hand-rolling
compensation on a path the platform owns, or waiting on a platform feature on
a path you own. Both directions read, locally, as diligent engineering.

## The measured case

The voice stack's browser door to Gemini Live cut itself off mid-reply on a phone. The
diagnosis-and-fix loop produced three *real* defects in one evening — an echo
threshold tuned for different hardware, a turn that ended on chunk arrival
instead of playback, a fallback timer leaking across turns — and each fix
helped, and none sufficed. A newly added flight recorder kept convicting the
next layer.

The actual answer had existed for a month, one directory away, with tests.
The same service's *terminal* door mapped a profile key to the platform's own
first-class config — `half_duplex: true` → `ActivityHandling.NO_INTERRUPTION`
— and its comment stated the whole theorem: *"then no echo can cut the voice stack off,
VAD or not."* Turn-taking on that path was the platform's job, configurable
in the session handshake. The browser door had never sent the config; every
client-side threshold was compensation for not using the path's own
mechanism. One field in the setup frame ended the failure class; the next
lived session recorded zero interrupts with echo demonstrably crossing the
old threshold and the server ignoring it.

The operator's directive, which is the pattern in one breath: *"Don't
reinvent. We're already on a couple of these paths — make sure you know these
paths exist before you try to invent new ones."*

## The pattern

- **Enumerate the paths, in a file the builder will read.** Which channels
  exist, and for each: who owns turn-taking and interruption — the platform's
  config, an adopted component's behavior, or your own seam. An unwritten
  enumeration is re-derived per incident, wrongly.
- **Name your path before writing any interaction machinery.** Echo gates,
  VAD thresholds, barge-in logic, liveness timers — before the first line,
  say which path this is and read its owner's contract. On an adopted path,
  the session/handshake config is the first place to look, not the last.
- **Check the sibling doors.** A service with two front ends to the same
  platform has usually solved the problem once already. The grep across
  siblings costs a minute; the evening of client-side fixes cost an evening.
- **The mirror direction binds too.** A path you own on purpose (because you
  control the transport end to end) keeps its hand-built machinery; porting
  the platform's assumptions onto it is the same crossing, reversed.

## Sharp edges

- Client-side fixes on the wrong layer are the dangerous kind of progress:
  each one is *real* (this evening's three all were), so the loop feels like
  it is converging while the class survives. If you have fixed the same
  interaction symptom three times in one session, stop and ask who owns the
  mechanism — the same tell as `one-authority-for-repeating-behaviors`, one
  layer up.
- Adopted-platform config is part of the adoption. "We use their realtime
  API" while sending a default handshake means you adopted their transport
  and reimplemented their product.
- The hygiene built along the way (a playback-anchored turn end, a flight
  recorder) may be worth keeping — as hygiene. The smell is only when it is
  load-bearing against a failure the path's owner would eliminate.

Related: `one-authority-for-repeating-behaviors` (same shape within one
codebase), `scan-prior-art-before-building-infra` (the sibling-door grep),
`wire-into-existing-flows` (an enumeration nobody reads is a standalone
artifact).
