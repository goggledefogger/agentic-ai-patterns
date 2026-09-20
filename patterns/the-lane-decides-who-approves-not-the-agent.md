---
type: pattern
date: "2026-09-07"
source: A household agent on a Raspberry Pi serving two people, the author and a family member, with the personal agent as the author's agent on the far side of a shared board. A family member's six asks about the property-reports page on 2026-09-07 became a task in a file nobody reads, and the fix was three keyed lanes on the board README of the agent board repo (the household-agent repo PR #286, the personal agent 08eee5e c944cda)
tags:
  - agents
  - permissions
  - multi-user
  - approval
  - shared-state
---

# The Lane Decides Who Approves, Not the Agent

When one agent serves two people, every ask has to land in exactly one of three
places: done on the next tick, done by a tap with nobody approving, or parked in
front of the one person who maintains the thing being changed. Which of the three
is a lookup on the request's key, made by code before the request reaches the
board and again after. The agent never judges whether an ask "seems small enough."

## The incident

A family member uses the household agent directly on Telegram. On 2026-09-07 she asked for six changes
to the property-reports page: newest first, archive the sold ones, a button to do
that herself, light mode, a listing link at the top, and a flood zone line in the
foundations check. The household agent did the right thing with the only surface it had. It
verified each ask against the live site, wrote task 137 into `TASKS.md`, and told
a family member that the author was on it.

Nothing reads that file to the author. The heartbeat script that would have surfaced it
is in no cron. He learned about the six asks from a transcript, hours later, and
a family member was left with a promise attached to nothing. This is `lawful-write-surface`
again: not a discipline problem, a missing surface. The handoff command only knew
how to open one kind of task, an evaluation, so an ask that was not an evaluation
had nowhere lawful to go.

## The Pattern

Key every request with a prefix, and let a closed table map the prefix to a lane:

| lane | key | who acts | approval |
|---|---|---|---|
| do it | `property-<slug>` | the runner, next tick | none, pre-approved by a spend row |
| bounded change | `site-change-<slug>` | a deterministic script, next tick | none, a tap applies it |
| decision | `request-<slug>` | a person | the one who maintains the thing, where the diff is |

Three properties make the middle lane safe with nobody in the loop. The verb list
is closed, `archive` and `unarchive` and nothing else, checked by both the sender
and the reader. The change is one line moved in a manifest, never a report body or
a rubric. And it is reversible by the same lane, so the worst outcome of a wrong
tap is a second tap. Either person may use it, because the list is both people's
data.

The decision lane goes to a human from the moment it is created: addressed to
the author, flagged `needs-human` before any agent has looked at it. The push he gets
says a decision is waiting and links to it. It does not say what the decision
should be, and it does not ask for a yes on Telegram, because a code change is
approved where the diff is (`consent-needs-the-diff`, `eyes-free-approval-channel`).
A tap on the push means "go ahead and open the work," never merge.

What comes back to each person is the receipt for their own ask, on the channel
they asked from. Never the other person's asks, never report contents, never
infrastructure status.

A recurring `request-` is the signal to grow the bounded lane by one verb. It grows
in the table, by a commit. It never grows because an agent decided an ask was close
enough to one it already knew.

## Why it works

- The agent has no judgment to get wrong. `a-model-never-picks-the-destination`
  applies to approval exactly as it applies to routing: which lane is also a
  question of who may be exposed to what, and a model call is the wrong place to
  answer it
- The "nobody approves" lane is not a hole in the tier model. It is a carve-out with
  three named conditions, and a fourth verb that fails any of them stays a
  `request-`. `a-tier-says-what-you-may-touch-not-what-others-may-see` puts every
  outward write in draft-and-ask, and this is the shape a legitimate exception takes
- The second person is a first-class requester. Every envelope carries who asked,
  and the receipt resolves that name to a chat by lookup. An envelope with no
  requester is refused rather than defaulted to the owner, which is how the owner
  stops being a firehose for someone else's work

## Watch-outs

- The rubric stays out of every lane. Anything that scores against household
  facts lives on one machine nothing networked touches, and a `request-` about it
  is a note to a person, never a task an agent performs
- `needs-human` needs a reader. Before this shipped the label had two writers and
  nobody watching it. A flag nobody reads is a file nobody reads with a shorter
  name
- The person-facing agent still owes an ack before the board post, so a board
  failure leaves the person informed rather than waiting on silence

## When to Use

Any agent with two or more human principals who can ask it for things, where some
asks are safe to just do, some are safe to do reversibly, and some change what a
maintainer owns. The tell that you need it is an agent that either stops for
approval on everything, which teaches the second person not to ask, or approves
things itself, which is the failure that gets an agent removed.

## Adjacent Patterns

- `coordinate-agents-through-shared-state.md` for the board the lanes ride on
- `the-sender-enforces-the-receivers-ceiling.md` for why the key is checked on
  both sides
- `a-tier-says-what-you-may-touch-not-what-others-may-see.md` for the visibility
  axis the middle lane carves out of
- `lawful-write-surface.md` for the diagnosis that preceded the fix
- `a-free-form-ask-lands-on-decided-rails.md` for the same move at the product
  layer
- `the-second-principal-is-invisible-to-a-reader-built-for-the-first.md` for what
  the second person's asks looked like from the operator's side that same day

## Source

Household boundary rule, Story 3.3 of the agent-to-agent channel epic, written
on the board README of the agent board repo (e45ea14) so both agents can
read it. The household agent side in the household-agent repo PR #286, the personal agent side in the agent's own repo
08eee5e and c944cda, 2026-09-07
