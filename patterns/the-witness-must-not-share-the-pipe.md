---
type: pattern
date: "2026-08-02"
source: Walk-and-talk phone bridge — the level meter that starved the speech recognizer (ux-debt 55)
tags:
  - code
  - reliability
  - observability
  - anti-pattern
---

# The Witness Must Not Share the Pipe: An Observability Tap on an Exclusive Resource Breaks What It Watches

A voice page's state label said "Listening" and users had stopped believing it, so a live input-level meter was added: a second microphone capture feeding an analyser, animating the orb with real voice level. Honest, glanceable — and it broke the listening. On Android the mic is effectively exclusive (a known, years-old Chromium bug: getUserMedia audio and SpeechRecognition cannot run together), so the meter's tap starved the recognizer: voice-activity detection *saw* the user speaking, woke the recognizer, and the recognizer received five seconds of dead feed while the user talked into it. The witness broke the witnessed — and for hours it masqueraded as three unrelated bugs ("says listening but doesn't hear", "no transcription", "wake worked but capture didn't").

The general shape: **adding an observability tap that acquires the same exclusive resource as the system it observes converts "no visibility" into "no function," silently.** Mic and camera handles, serial ports, single-connection databases, file locks, GPU contexts, debug probes on contended buses — anywhere acquisition is exclusive or quota-bound, a monitor that acquires is a second tenant, not a bystander.

## The Pattern

1. **One owner per exclusive resource, as an architectural rule.** Decide which component owns the handle. Every other consumer — including monitoring — gets its signal *derived from the owner* (events, counters, interim output), never from a parallel acquisition.
2. **The most honest proof is the owner's own output.** The recognizer's interim words prove hearing better than any side-channel meter, because they come from the thing doing the work. Prefer proof-of-work over proof-by-parallel-measurement.
3. **Guard the class with a test, not a memory.** After the fix, a test asserts zero secondary acquisitions while the owner runs (stub the acquisition API, count calls). The rule survives the next contributor who wants a meter.
4. **Suspect the tap when the primary gets flaky right after observability lands.** The failure signature is temporal: the system worked, a witness was added, the system "developed" intermittent deafness/blindness. Check the tap before theorizing about the primary.

## Corollary

If the observed system is so opaque that a parallel tap feels necessary, that opacity is the bug to fix — surface the owner's internal signal (events, partial results, level callbacks) rather than standing up a competing consumer.

## Related

- [[latched-state-needs-a-reconciler]] — same day, same system: state and signal must reconcile, but the reconciling signal must not cost the resource.
- [[decorative-gate]] — the inverse failure: a control that observes nothing; this one observes truly and destroys the thing observed.
- [[verify-by-exercising]] — the tap passed every state-machine test; only exercising on the real device revealed the contention.
