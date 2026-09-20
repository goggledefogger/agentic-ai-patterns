---
type: pattern
date: "2026-08-03"
source: The work vault, 1:1 prep brief generated 2026-08-03. A person file's newest note recorded "hoping to fill it internally at 0.5 FTE, confirmation expected this week," dated two months earlier. The brief rendered it as accomplished fact and built four of six beats on top of it.
tags:
  - documentation
  - drift
  - second-brain
  - verification
  - anti-pattern
---

# Silence Is Not Confirmation: An Append-Only Log Has No Present Tense

A person file in a second brain accumulates dated notes: a 1:1 summary, a decision, a Slack thread worth keeping. It is an append-only log, and it is the right shape for the job. Then something reads it to answer a different question — *what is true about this person today?* — and the only heuristic available is recency. The newest note becomes the current state.

That works until the newest note is a **forecast**. "Confirmation expected this week." "Planning to start next PI." "She'll decide by Friday." Written accurately, at the time, about a near future. Then nobody writes the follow-up, because the follow-up is only interesting if the answer was surprising. Months pass. A reader arrives, takes the newest note as state, silently drops the forecast verb, and reports a plan as an accomplishment.

Concrete instance (the work vault, 2026-08-03). A designer's file ended with a June 2 note: a colleague was *hoping* to fill a role internally with him at 0.5 FTE, *confirmation expected that week*. No note after it. A generated 1:1 brief opened with "now that you're half on a new project," made it the premise of four of six agenda items, and asked follow-up questions that only make sense if it happened. Nothing in the vault said it happened. Nothing said it didn't. The same draft also assigned him a task that the vault's `TODO.md` assigned to two other people, because the person file named the task without naming an owner.

## The Problem

This is not stale data. Every note in the file is still true. It is a **missing layer**: the log records what was *said*, and nothing records what *is*. The reader synthesizes the second from the first, and the synthesis has no error bars.

Three properties make it hard to catch:

- **It reads as knowledge, not as a gap.** The brief was specific, dated, and cited its source correctly. Confidence is inherited from the note's precision, not from its currency.
- **Silence is doing two jobs at once.** No follow-up note means either "it happened, unremarkably" or "nobody checked." Those are opposite, and the log cannot distinguish them — but a reader defaults to the first, because logs mostly record things that went as planned.
- **The stale entry is the newest one.** Every staleness instinct is calibrated on old content. A forecast is stale the moment its horizon passes, which can be days after it was written, while it still sits at the bottom of the file looking like the latest word.

Distinguish it from the neighbors. [`stale-pointer-asserts-confidently.md`](stale-pointer-asserts-confidently.md) is a fact copied into another repo that rots when the original moves — there, the recorded claim became false. Here the recorded claim is still true; it was always a forecast and still is. [`self-reporting-staleness-check.md`](self-reporting-staleness-check.md) detects broken links and past-due `revisit_by` dates — a forecast note has neither, so that scan runs clean over it. [`latched-state-needs-a-reconciler.md`](latched-state-needs-a-reconciler.md) is the same shape at runtime: a state set by one event and never cleared by the event that should have cleared it, with nothing comparing the claim against reality.

## The Pattern

1. **Give entity files a pinned state block, separate from the log.** One `## Current` section directly under the H1, holding only what is true now: live commitments, open questions, what the next conversation is for. The dated notes stay below, append-only, untouched. Readers consult the block; the log is evidence they can drill into. The separation is the whole fix — it creates a place where "nothing changed" has to be asserted rather than inferred.

2. **A forecast note is not done until it names its resolution owner and date.** "Confirmation expected this week" is half a record. "Confirmation expected this week — if nothing here by June 10, ask him directly" is a complete one, and it converts into a `revisit_by` the staleness scan can already see.

3. **Readers must treat forecast verbs as questions, never premises.** *Expecting, hoping to, planning to, leaning toward, will confirm by, pending* — when one of these is the newest word on a topic and no later note resolves it, the correct output is "did this happen?" at the top of the document, not a present-tense claim in the middle of it. Load-bearing unconfirmed claims get a visible warning, not a silent promotion.

4. **Ownership lives in the task system, not the entity file.** A person file records that a task was discussed. It routinely omits who owns it, because in the room that was obvious. Cross-check every owed item against whatever holds assignments, and let that source win — otherwise the person you are reading about inherits the task by proximity.

5. **Add the forecast verbs to the corpus hazard rules.** They are greppable, which makes this the rare judgment failure with a mechanical detector. In `.staleness-rules.txt` (see [`self-reporting-staleness-check.md`](self-reporting-staleness-check.md)):

   ```
   confirmation expected  => unresolved forecast; check for a later note or ask
   expected to confirm    => unresolved forecast; check for a later note or ask
   will confirm by        => unresolved forecast; check for a later note or ask
   pending confirmation   => unresolved forecast; check for a later note or ask
   ```

   Noisy by design. A hit is not a defect, it is a prompt to look — and the ones that resolved cleanly cost one glance each.

## Why It Works

- **A state block makes staleness visible as emptiness.** An out-of-date `## Current` looks wrong on sight in a way that an old dated note never does, because a dated note is *supposed* to be old.
- **Naming the resolution owner closes the loop at write time**, when the cost is one clause and the context is still in hand. Every later fix costs a search.
- **Verb-level detection catches the class, not the instance.** You cannot enumerate which forecasts will go unresolved, but you can enumerate the words that introduce one.

## Watch-outs

- **A `## Current` block that nobody updates is worse than none**, because it claims to be current. Tie its refresh to an existing ritual (post-meeting processing, weekly review) rather than to intention — see [`wire-into-existing-flows.md`](wire-into-existing-flows.md).
- **Do not resolve a forecast by inference.** "It was probably fine" is the exact move this pattern exists to stop. Unresolved means ask.
- **The absence of a hazard-rule hit proves nothing.** Plenty of forecasts are phrased without a flagged verb. The rules catch the common shapes, the `## Current` block catches the rest.

## Related

- [`stale-pointer-asserts-confidently.md`](stale-pointer-asserts-confidently.md), a recorded fact that became false; here the record stayed true and the reader supplied the falsehood.
- [`self-reporting-staleness-check.md`](self-reporting-staleness-check.md), the scan this pattern contributes new rules to.
- [`latched-state-needs-a-reconciler.md`](latched-state-needs-a-reconciler.md), the runtime sibling: state set by an event, never cleared, nothing reconciling claim against reality.
- [`living-doc-refresh-ritual.md`](living-doc-refresh-ritual.md), how the state block stays true once it exists.
- [`doc-warning-preamble.md`](doc-warning-preamble.md), what to do with a load-bearing claim you could not verify.
