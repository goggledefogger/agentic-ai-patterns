---
type: pattern
date: "2026-08-02"
source: Walk-and-talk phone bridge, one live walk day (ux-debt 49–53) — the "banana" dead-mic incident
tags:
  - code
  - reliability
  - voice
  - anti-pattern
---

# Latched State Needs a Reconciler: A Flag Set by Events Will Eventually Be Set Forever

A voice page tracked "am I speaking?" as a counter: +1 when an utterance starts, −1 when its end event fires. The mic was (correctly) gated shut while speaking. Then Android's speechSynthesis did what it routinely does — fired `onstart` and never fired `onend`. The counter stuck at 1, the mic stayed gated for 30 seconds, the label said "Listening" the whole time, and the walker's words went nowhere. The gate was right; the state feeding it was a lie that could never self-correct.

The general shape: **any latched state (a flag, counter, lock) that is set by one event and cleared by another will eventually latch forever**, because somewhere, sometime, the clearing event is lost — a platform bug, a killed process, a superseded handler, a backgrounded page. The more critical the resource that state gates, the worse the wedge.

## The Pattern

Two defenses, both, not either:

1. **Guarantee the release locally.** Every acquire gets exactly one release from *whichever* signal arrives first: the normal end event, the error event, or a duration-based backstop timer sized to the work. A `closed` flag makes them idempotent. (One extra: only release what was actually acquired — an end without a start must not push the counter negative.)
2. **Reconcile the claim against observable reality, globally.** A cheap watchdog compares what the state *claims* against what can be *observed*: "speaking > 0" vs. "is any audio element actually unpaused, is the synth actually speaking or pending." N consecutive seconds of contradiction → log the discrepancy (with the claimed value), force-reset, and re-open the gated resource. The watchdog catches every wedge class you didn't enumerate in defense 1 — including the ones added by next month's code.

Defense 2 is what makes the difference. Defense 1 fixes the failure you found; the reconciler fixes the family.

## Corollaries from the same day

- **A new gate inherits every way its input state can wedge.** The mic gate was added in the morning (correctly, to stop playback ducking); by afternoon it had turned a pre-existing, previously-harmless bookkeeping leak into a dead mic. Adding a consumer to a state variable is the moment to audit how that variable can stick.
- **A state label trusted once and wrong once is trusted never again.** The durable fix for "it says Listening but I don't believe it" was not a better label — it was rendering *observable* evidence (live input-level animation from an analyser tap, interim transcription as it forms). Show the signal, not the claim.

## Related

- [[decorative-gate]] — a control that can't refuse; this pattern is its sibling: a control fed by state that can't recover.
- [[convergent-standup-sloppy-quits]] — same philosophy at process scale: never depend on the previous thing having ended cleanly.
- [[verify-by-exercising]] — the tests all passed while the behavior was broken, because two tests *specified* the broken design (always-listening re-arms) as correct. A green suite cannot catch a wrong spec; when a live report contradicts passing tests, suspect the spec the tests encode.
