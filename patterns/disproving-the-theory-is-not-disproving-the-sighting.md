---
type: pattern
date: "2026-08-05"
source: The household-agent repo — "I don't see it done in the UI"; the ledger proved the download had finished two minutes *earlier*, so two agents in a row filed it as a non-event
tags:
  - debugging
  - anti-pattern
  - agents
  - ux
  - verification
---

# Disproving the Theory Is Not Disproving the Sighting

A user report is usually two things fused into one sentence: **an observation**
("the UI said stopped") and **a theory** ("...and my complaint is what fixed
it"). The theory is the part an agent can check fastest, because theories make
timestamp claims and timestamps are cheap. Refuting it feels like closing the
ticket. It closes nothing — the observation is still standing there, unexamined,
and it was the real bug.

## The Problem

A user asked for one TV episode by voice. Six minutes later the assistant said
it was downloading. Two minutes after *that*:

> **User:** I don't see it done in the UI

The assistant checked its download ledger, the filesystem, and the media
server, found the file complete and indexed, and replied "it just finished."
Later, a second agent reviewing the day's transcripts read the same ledger,
saw the download had completed at **18:00:42** — a full **110 seconds before**
the user typed at 18:02:32 — and wrote it up as *handled cleanly*. The user's
"and then it showed up right after I said something" was filed as coincidence,
with a timestamp to prove it.

The timestamp was right. The conclusion was wrong.

What the user had actually been looking at was a **second download lane**: a
torrent for the same episode, dead on arrival at 0 seeds, that the interface
was still rendering as an active multi-gigabyte **stopped** download with a
Resume button. It cleared four minutes and twenty-two seconds after the file
was already watchable, when an unrelated five-minute cron swept it — forty-one
seconds after the assistant's reply, which is precisely why it felt causal.

Both agents made the identical error, and it is worth naming exactly:

> The user said *"in the UI."* Between them they checked the status file, the
> outcomes ledger, the media server's item list, and the filesystem. **Neither
> queried the UI.**

Every source consulted was authoritative for *"did the file download?"* — a
question nobody had asked. The evidence was real, current, correctly read, and
aimed at the wrong question, which is the most expensive kind of evidence
because it terminates the search with confidence.

## The Pattern

**Split the report before you investigate it. Verify the observation on the
surface where it was made; treat the theory as a separate, lower-priority
claim.**

1. **Look at the named surface first.** If the user says "the UI," open the UI
   — or the endpoint behind it. If they say "the email," read the email. The
   surface is not a rhetorical detail; it is the only place the reported
   artifact is guaranteed to exist. A backend that disagrees with a frontend is
   a finding, not a refutation.
2. **A correct timeline refutes causation, never existence.** "It was already
   done when you complained" and "you saw a stopped download" are both true
   here. When the timeline rules out the user's *mechanism*, that is the moment
   the observation becomes more interesting, not less — something made a
   competent person read success as failure.
3. **Ask what else was on screen.** The bug is often not the object you are
   tracking but an adjacent, staler one with a scarier label. Enumerate
   everything the surface renders for that entity, not just the record you have
   been reasoning about.
4. **"Right after I said something" usually means a poll.** Users cannot see
   cron ticks. When an effect lands suspiciously close to a message, look for a
   periodic sweep whose interval brackets the gap before concluding
   coincidence — and note that the *maximum* staleness that sweep permits is a
   number nobody has ever costed.

## When It Shows Up

Any report of the form "X happened right after Y," "it fixed itself when I
did Z," or "it says the wrong thing" — especially when the agent holds a clean
log that appears to settle it. Also every dashboard/backend split, where the
store is right and the view is stale, and the user can only ever see the view.

Distinct from [[plausible-cause-ends-the-search]], where memory supplies an
explanation *before* live evidence is read. Here live evidence *was* read, was
accurate, and answered a question the user had not asked — a failure of
question-framing rather than of freshness. Compare
[[freshness-axis-must-match-the-question]]: same family, one axis over.

## Related

- [[plausible-cause-ends-the-search]] — an explanation that fits, and isn't the cause
- [[freshness-axis-must-match-the-question]] — right data, wrong axis
- [[a-fast-answer-is-a-suspect-answer]] — speed as a smell
- [[the-losing-lane-keeps-rendering]] — the bug this report was actually about
- [[ask-challenge-verify-correct]] — the user pushing back is signal

## The Rule of Thumb

When a user tells you where they were looking, look there. Their explanation
may be wrong; their sighting almost never is.
