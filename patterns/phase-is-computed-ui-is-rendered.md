---
type: pattern
date: "2026-08-02"
source: Walk-and-talk phone bridge — the day of lying "Listening" labels (ux-debt 56)
tags:
  - code
  - ui
  - state
  - anti-pattern
---

# Phase Is Computed, UI Is Rendered: Status Hand-Set From Permissive Flags Will Lie

A voice page's status label said "Listening" while the assistant was talking, again during a multi-second render window, and again through an entire think with the mic dead. Three patches in one afternoon, each adding another hand-set label at another call site — and the user finally named it: "this feels like patchwork." The root: a dozen places wrote the label from flags that *permit* listening (hands-free on, not muted) rather than facts that *prove* it (a recognizer actually running). Even the designated truth function answered "could we listen," not "are we." Every consumer of that answer — the label, the connection gate, the tap handler — inherited the lie, and the tap handler's version had behavioral teeth: "hands-free means tap-to-mute" assumed the mic was always on in hands-free, so a user tapping a resting mic to *resume* listening silenced it instead.

The tell that you're in this failure mode: **the same class of bug needs a third patch.** At that point the bug is the representation, not any call site.

## The Pattern

1. **One `phase()` function computes the single current phase from observable primitives** — is the engine running, is audio actually playing, is a request in flight — in an explicit priority order. Never from flags that merely permit. (Priority encodes truth: a genuinely live ear outranks "a reply is being prepared," because then "listening" is simply true.)
2. **One table maps phase → presentation; one `render()` writes the UI.** No other code touches the status label or state classes. Every event handler ends with `render()`.
3. **Commands decide off the computed phase too.** A tap/click/keypress dispatches on `phase()`, not on its own reading of the flags — otherwise the UI and the controls disagree about what state you're in, which is how "tap to resume" becomes "mute."
4. **Derived views for other consumers wrap the same machine.** A health report, a remote gate, an accessibility announcement — each is a projection of `phase()`, never a second derivation.

## What it buys

- A new lying-label bug becomes structurally impossible to fix wrong: you fix `phase()` once and every consumer heals.
- New states (a "rendering" window, a "resting" ear) are one table row, not N call-site audits.
- Tests assert phases and rendered words directly, so the honesty is pinned, not remembered.

## Related

- [[latched-state-needs-a-reconciler]] — the input side of the same day: the primitives `phase()` reads must themselves be reconciled against observable reality.
- [[the-witness-must-not-share-the-pipe]] — where the observable proof must come from (the owner's own output).
- [[decorative-gate]] — the shared ancestor: a claim ("passed", "listening") that nothing computed.
