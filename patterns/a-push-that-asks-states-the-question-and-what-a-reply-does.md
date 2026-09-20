---
type: pattern
date: "2026-09-11"
source: The personal agent's Telegram pushes to the author. A push told him to "answer on the issue, or tell the personal agent"; he asked how. A later push told him to "reply with your answer"; he asked what the question was. The chat those pushes land in files any unrecognised text as a voice note.
tags:
  - notifications
  - human-in-the-loop
  - agents
  - ux
---

# A Push That Asks States the Question, and What a Reply Does

A notification that wants something from a person carries two facts in its last
lines: **what the question is**, and **what a reply here does and how soon** —
or that nothing here accepts a reply, and what a reply would turn into instead.
A push missing either one is a working door nobody has labelled, which is
indistinguishable from no door.

## The incident

The personal agent pushes board activity to the author's phone. On 2026-09-11 a new push kind went
live for requests addressed to him. It read, in full: a title, a link, and
"This is a decide-or-build ask, not a yes/no: answer on the issue, or tell the personal agent."
The author: *"it says answer on the issue, how should i?"* A reply to that push
matched nothing, so the chat would have filed his answer as a voice-note
transcript — the one place an answer could go and never be read as one.

Once a reply worked, the next push said "Reply here with your answer." The author:
*"To give my verdict, it's hard to know what the question is."* There were two
questions at two moments — *should this be built?* when the request arrived,
and *does it look right?* once the personal agent had shipped it — and neither was stated. The
second moment had no push at all; he learned the work was done because a session
told him.

## The Pattern

Every push kind ends with a fixed contract line, chosen by kind:

| the push | last lines |
|---|---|
| a request has arrived | *The question: build it, change it, or drop it?* Reply here with your call — it lands on the issue as your comment within a minute. Or comment there directly. |
| the request has shipped | *The question: does it look right?* Reply "close" to close it, or say what's off — it lands on the issue within a minute. |
| an approval | Tap Yes or No, or reply here with one word — picked up within a minute. |
| answered / gone quiet / unclaimed | Nothing to answer. A reply here files as a note, not an answer. |

The delay is stated because the pickup is a timer, not a listener. The
"shipped" push exists at all because a decision has a second moment, and the
first push cannot carry a question that has not been asked yet. And the
envelope that opened the request carries the same question in its own
`done-when`, so the issue body agrees with the phone.

## Why it works

- `a-gate-can-block-but-cannot-speak`: the reply path was built and working
  before it was named, and being unnamed it did not exist for the one person it
  was for
- The chat's default is capture. In a channel where unrecognised text becomes
  a note, an unstated contract is not neutral: a real answer silently becomes
  the wrong kind of record
- Stating the question is what makes a one-word reply lawful. "close" means
  something only against *does it look right?*; against *build it or drop it?*
  it is ambiguous, and the decider should never have to guess which question a
  push is asking

## Watch-outs

- The reply matcher keys on the push's first line. The contract lines can be
  reworded freely; the opening phrase cannot, or the door closes silently
- The push quotes the issue title, and when the title is the asker's exact
  words it truncates mid-sentence. Lead with who asked and a short form; keep
  the exact words on the issue
- "Within a minute" is a promise about a timer on another machine. If that box
  has not synced the code that reads replies, the push is telling the truth
  about a door that is not there yet — deploy before you announce
