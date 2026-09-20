---
type: pattern
date: "2026-09-11"
source: The personal agent's agent board. A feature a family member asked for shipped the same day, found three houses it now failed, and left the verdict flip to the author — as a comment on the issue the personal agent then closed. Four days later nothing had raised it and the reports still contradicted themselves.
tags:
  - agents
  - human-in-the-loop
  - state
  - notifications
  - shared-state
---

# A Closed Ticket Is Not a Home for an Open Decision

Finishing the work and leaving a decision behind are two different acts, and the
second one needs its own home. A decision only a person can make never lives
solely in something closed, terminal, or completed — not a comment on the issue
being closed, not a prose note in a config file, not a field in a state file.
Those are the places nothing looks again.

## The incident

On 2026-09-07 a family member asked for a flood-and-tsunami line in every property
report's foundations check. The personal agent shipped it the same day and backfilled all
twenty-five reports. The check found three houses that failed it. The template's
own rule says a defeated foundation makes the verdict FAIL, but flipping a
verdict a research pass had written was a person's call, so the personal agent did the correct
thing: he wrote "Left open for the author, deliberately" — in a comment on the issue
he closed in the same breath.

Nothing re-raises a closed issue. Four days later a published report still read
`FAIL — FEMA Zone AE` on its hazard line and `Outcome: PENDING` at the top, and
nothing was ever going to mention it. The same board held a task flagged for a
person, open twenty-four days, pushed to the author's phone exactly once on the day it
was flagged and never again: the reader that announces once is the same bug in a
different coat.

## The Pattern

A pending human decision gets an **open** item with an owner, on a surface that
re-raises it for as long as it waits. On the personal agent's board that is an issue addressed
to the person and flagged from creation; the session-start sweep lists every
such issue, oldest first, at every start until it closes. The notifier that
fires once per state stays exactly as it is — its job is to announce, not to
remember.

The rule at the moment of closing: if any part of the outcome is "someone has to
decide," that part is filed before the close, as its own thing, in the deciding
person's name. The work closes; the decision opens.

## Why it works

- `known-broken-must-not-page`: an unresolved condition must stop paging and
  stay visible in something regenerated every run. A closed ticket is the
  opposite of regenerated; a session-start sweep is exactly that
- `silence-is-not-confirmation`: a forecast line in a terminal artifact rots
  into a false positive — "left open for the author" read, four days on, like a
  thing that had been handled
- `latched-state-needs-a-reconciler`: the verdict was a latch set by one event
  and never cleared by the other; the reconciler is the sweep, not trust that
  someone will remember

## Watch-outs

- The re-raise belongs on the deliberately-read surface, never the push
  channel. Re-pushing a known-unresolved item trains the person to ignore the
  channel; listing it at session start costs nothing and cannot be muted by
  habit
- A flag with one writer and no reader is prose. `needs-human` had a reader
  that fired once; a reader that fires once is a reader for one day
- The item must state the question, not only that one exists — see
  `a-push-that-asks-states-the-question-and-what-a-reply-does`
