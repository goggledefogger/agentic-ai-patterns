---
type: pattern
date: "2026-08-02"
source: The personal agent — walk-and-talk phone bridge posted the author's half-finished sentences as completed turns, found by ear on a live walk 2026-08-02 (ux-debt 57)
tags:
  - architecture
  - reliability
  - voice
  - anti-pattern
---

# An Event Is Not a Cause

A speech recognizer fires `onend`. The page commits the transcript it has been accumulating and sends it as the user's turn. That reads as obviously correct — the recognizer stopped, so the person stopped talking, so the turn is over.

It is wrong, and the way it is wrong is worth naming, because the same substitution is available in every event-driven system. `onend` is a fact about a recognizer's lifetime. "The walker finished a sentence" is a claim about a person. Reading the second off the first means the page speaks for the user, using words they were still in the middle of saying.

## The Problem

On a walk, the author's turns started arriving cut off mid-clause: *"okay so the other thing is I want"*. Not garbled — truncated, and submitted anyway, as though he had stopped there.

`onend` fires for at least three unrelated reasons, and the code could not tell them apart:

1. **The walker actually finished.** The only one that justifies a send.
2. **The platform timed out.** Android kills an idle recognizer in one to three seconds — routinely mid-thought, while someone is choosing a word.
3. **The page killed the mic itself**, to start playing a reply. Android ducks media while a recognizer is live, so playback begins by aborting capture.

Case 3 is the one that hurts most, and it is entirely self-inflicted: **the system interrupts the user, then submits the fragment it caused.** The diagnostic log had been saying so for hours — five `recog-error aborted` inside one minute, each followed by a send — but nobody read it until a human complained by ear.

The tell is grammatical. The handler was named for a transition and used as a decision.

## The Move

**Name the cause, then act on the cause.** The send is triggered by the walker going quiet — a silence timer armed by their own most recent word — and a session ending never triggers it. That one reframing removes cases 2 and 3 together, with no special case for either.

**Make self-inflicted stops legible to their own handlers.** Every page-initiated abort now stamps the dying object, so its `onend` can tell "I was killed" from "I ended". A bare `abort()` anywhere else in the file is indistinguishable from the user finishing — which is exactly how this shipped.

**Hold across the boundary instead of committing at it.** When capture stops for the system's own reasons, the partial input stays uncommitted and visible; the next session continues the sentence. See [[hold-vs-drop-on-interrupted-input]].

**And write the invariant where a reader meets it before the code does.** Stated plainly — *a lifecycle event of our own capture engine is never sufficient justification for putting words in the user's mouth* — the original wiring is wrong on sight. It took a live walk and a log to find. It should have taken a reading.

## Beyond Voice

The shape is general; voice is only where it was expensive. Any handler that treats "the thing that happened" as "the reason to act" is one platform quirk from the same bug — a connection close read as a clean logout, a process exit read as job success, a file-watcher event read as an edit the user made. In each, the event is real and the inference is unearned.

Ask of any handler: **what fires this, and is every one of those reasons a reason to do what I am about to do?** If the answer needs a qualifier, the trigger is the wrong one.

## Related

- [[latched-state-needs-a-reconciler]] — state set by a transition latches forever when the clearing event is lost; the same event-as-truth mistake, held rather than acted on.
- [[self-report-must-not-end-what-it-reports-on]] — budget on silence, not on the subject's state.
- [[hold-vs-drop-on-interrupted-input]] — what to do with the words you interrupted.
- [[half-duplex-turn-taking]] — the other half: not interrupting in the first place.
- [[a-fake-can-only-fail-the-ways-you-have-seen]] — why the test suite stayed green through all of this.
