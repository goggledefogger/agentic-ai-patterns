---
type: pattern
date: "2026-08-01"
source: Walk-and-talk phone bridge, live on a real walk. A "still working" status line shared the reply path, so speaking it ran the reply's teardown — the thinking state cleared and the reassurance hum stopped, leaving the rest of a long turn silent. Found by ear, not by any test.
tags:
  - ux
  - voice
  - state-machines
  - anti-pattern
  - observability
---

# A Self-Report Must Not End What It Is Reporting On

A system that narrates its own progress has to emit that narration through the
same channel it uses for real output. That channel almost always carries
**completion semantics** — it clears state, releases locks, stops the spinner,
re-arms the input. So the act of saying *"still working"* runs the teardown for
*"done working"*, and the status update silently ends the thing it was
describing.

The tell: the report is correct, and the system is quieter or emptier
immediately after it.

## The failure

A voice walk hums softly while thinking, so the walker knows it is alive. A
guard was added to speak *"this is taking a while, still on it"* when the wait
ran long.

Speaking it went through the ordinary `speak()` path, whose end-of-utterance
handler does what finishing a reply should do: drop the thinking class, stop the
hum, restore the idle label, re-arm the microphone. All correct for a reply.
Catastrophic for a status line — the walk went **silent for the remainder of the
turn**, which is worse than the unbounded hum the guard was written to fix.

Nothing errored. The words were true. The state machine simply learned the turn
was over from an utterance whose entire purpose was to say it was not.

## The pattern

**Anything a system says *about itself* must hand its state back exactly as it
found it.** Five requirements, each of which was violated in turn — the first
three by the original defect, the last two by successive attempts to fix it:

1. **Mark the utterance.** The speech path takes a flag (`isStatus`) and skips
   the completion teardown for it. Without a marker the two are indistinguishable
   at the point where it matters — the handler.
2. **Restore, don't just skip.** The status line still legitimately interrupts
   the hum while it speaks. When it finishes it must **resume** the hum and the
   thinking state, not merely avoid clearing them.
3. **Give the subject its own clock.** Elapsed time was read from the hum's
   timer, which the announcement stopped — so after speaking, "how long" reset
   and every later status answer lied. The turn needs a clock that survives being
   spoken about.
4. **Budget the report on silence, not on the subject's state.** Two successive
   versions of this guard looped, and both failed the same way: the first keyed
   its "already announced" flag to the hum (which its own speech stopped), the
   second to a turn counter (which the *fix* incremented when it resumed the
   hum). Any budget derived from state the announcement mutates will eventually
   reset itself. **"Nothing has happened for N seconds" cannot be reset by
   accident** — and the announcement itself counts as something happening.
5. **Never emit while already emitting.** Speech engines, toast stacks and log
   sinks all queue. A guard that fires while its previous output is still playing
   does not overwrite it, it *stacks* — turning one reassurance into a barrage.
   Check the busy flag before speaking, not only the timer.

Requirement 3 generalises furthest: **a metric must not be stored in the thing
whose interruption it is meant to measure.** Requirement 4 is its scheduling
twin: **a budget must not be stored there either.**

## Where else this shape appears

- A progress log written through the same writer that closes the file on flush.
- A heartbeat sent over a connection whose send path resets the idle timer, so
  the heartbeat itself masks the death it was added to detect.
- A "still running" notification that touches the job's last-activity field,
  making a hung job look healthy forever.
- A health check that consumes the queue item it is checking for.
- Rendering a spinner update through the same call that dismisses the spinner.

In every case the reporting mechanism is coupled to the lifecycle it observes,
so observing changes the observed. The fix is always the same shape: an explicit
marker at the boundary, restoration afterward, and a clock the report cannot
touch.

## Verification

**A unit test will not catch this.** The status function returns the right
string; the state machine is correct in isolation; each half passes. The defect
lives in the composition, and it presents as *absence* — a sound that does not
come back.

So the check has to assert the **after** state, not the output:

- After a status utterance completes, is the thinking state still set?
- Is the ambient signal audible again?
- Does elapsed time still read from the true start of the turn, not from the
  announcement?

It was found because a human noticed something missing. Absence is the hardest
class of bug to test for and the easiest to hear, which is an argument for
keeping a person in the loop on anything ambient.

## Adjacent Patterns

- **`monitor-cannot-see-its-own-exhaust.md`** — the closest sibling: there the
  monitor mistakes its own output for signal; here the reporter destroys the
  signal by reporting.
- **`count-the-source-not-the-survivors.md`** — both are failures where the
  instrument changes the population it measures.
- **`unrun-checks-read-as-passing.md`** — silence mistaken for health, arrived at
  from the other direction.
