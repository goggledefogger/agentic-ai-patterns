---
type: pattern
date: "2026-09-11"
source: A household agent receiving asks from a family member and the author about the property reports. Twice in four days the first hop decided an ask was too big and answered with something smaller, and the original ask never reached the one person who could size it.
tags:
  - agents
  - handoff
  - multi-user
  - routing
---

# Never Narrow at Intake

The first agent to receive an ask records it in the asker's own words, dated,
and sends it up. It does not judge feasibility, size, or cost; it does not offer
a smaller version; it does not read code to decide whether it could. Only the
person who maintains the thing narrows an ask, and they do it where the diff is.
An agent that narrows at intake has not routed the ask — it has replaced it, and
the replacement carries no trace of what was asked.

## The incident

On 2026-09-07 a family member asked the household agent to "delete or archive any reports where the
property is either pending sale or sold." The household agent answered, reasonably, that the
reports "don't automatically track when a listing goes pending or sells," and
proposed manual archiving instead. The manual lane was built that day. The
tracking half — the part a family member would have felt most — never became a task
anywhere, and nobody upstream knew a narrowing had happened, because the
narrowed version is all that travelled.

Four days later, with a never-narrow rule deployed, the author sent a test ask. In a
session that still remembered a timed-out handoff, the household agent read the site repo's
README to judge whether she could do it and replied, "No, mate, I can't do that
from here." The routing line said "NOT yours"; the model read *not yours* as
*cannot*, then went looking for evidence and found some.

## The Pattern

At the first hop, the only judgment is *which lane*, and the lane is a lookup
(`the-lane-decides-who-approves-not-the-agent`). Everything else is carried, not
decided:

- The task field is the asker's exact words with the date, so nothing is
  paraphrased before the maintainer reads it
- Feasibility, size, and "would a smaller thing do" are the maintainer's calls,
  made against the actual code, and the answer comes back as a decision on the
  record — never as the agent's opinion in chat
- "It can't do that" is not an answer the first hop is allowed to give; the
  lawful reply is that it has gone up
- A repeated ask means the last handoff did not land. Re-fire it; do not
  analyse why the person is repeating themselves

## Why it works

- `a-model-never-picks-the-destination` extends to scope. Deciding an ask is
  too big is picking a destination for the part that got dropped: nowhere
- The narrowing is invisible to everyone but the agent. The asker hears a
  reasonable answer; the maintainer sees a reasonable task; the gap between
  them is in no record
- A polite refusal is the ask dying with a receipt. It is worse than silence
  because it closes the loop for the asker while leaving the work undone

## Watch-outs

- Wording that reads as a capability claim gets treated as one. "NOT yours"
  became "cannot"; "not yours to build, yours to send up" did not. Say what the
  agent *does*, not only what it is not
- A warm session is the normal case. The agent remembers the failed attempt and
  reasons about it instead of routing; the rule has to name the repeat
  explicitly, because "I already tried" is exactly when a model starts judging
- A routing line cut down to fit a size budget can lose its imperative and
  still pass every unit test. Test the behaviour with a real message from a
  real chat, in a warm session, and read the reasoning the model logged
- The fix for "she went and read the code" is not to forbid reading code in
  general; it is to say that on this kind of ask the handoff command *is* the
  answer, so there is nothing to research
