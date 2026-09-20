---
type: pattern
date: "2026-09-07"
source: The household-agent repo, the Pi's transcript reader (a script). A family member's three inbound messages on 2026-09-07 were in the store, parsed, and counted, then dropped at print time because both render loops tested `role == "OWNER"`. The session that reviewed the day rebuilt her asks from the household agent's replies without noticing. Fixed in the household-agent repo ebb02b4
tags:
  - agents
  - multi-user
  - observability
  - anti-pattern
  - testing
---

# The Second Principal Is Invisible to a Reader Built for the First

When a system gains a second person, every reader written while there was one
keeps working, keeps counting, and shows you only the first. The loss is not at
ingest and not a bad value. The data is present and correct, right up to the line
that decides what to print.

## The incident

The household agent has had two users for months. The transcript reader that every catchup
runs resolves who sent each message from the session's user id, so a family member's
turns were labelled `SECOND` correctly, counted correctly, and appended to the
message list correctly.

Then the print loop ran, and it had two branches for humans: one for `OWNER`,
and none. Her three messages fell through. The session header said `owner:3`,
because the counter's label was a hardcoded string too. The transcript showed
three replies from the household agent and nothing they were replying to.

The morning's review session read that output, saw the household agent enumerate four of
a family member's asks in its own reply, and reconstructed the conversation from the
answers. It wrote a correct account. It did not notice that half the
conversation was missing, because a transcript with only one side of it still
reads like a transcript.

The test file had a fixture with one user in it, the author. Eleven tests were green.

## The Pattern

**A reader written for one principal encodes that principal as a constant, and
the constant survives every later change that adds a second.** The parser gets
updated because parsing fails loudly. The renderer does not, because a renderer
that skips a row fails silently, and the rows it keeps look complete.

The shape to look for:

- a label compared to a literal where a set of known labels exists a few lines up
- a counter whose name is a person rather than a role
- a fallback that maps "any known id" to the first person instead of to the id it
  matched
- a fixture that only contains the case that passes

The fix is one set, built from the same map that resolves ids to names, used
everywhere a human label is tested. The header names the person it counted. The
fallback returns the name for the id it found. And the fixture gets a second
person, because a population filtered to the passing case cannot fail on the bug
(`count-the-source-not-the-survivors`, one layer up).

## Why it works

- The map of who is a human already exists, because the parser needed it. Deriving
  the render test from that map means a third person appears in transcripts the
  day they are added to the map, with no render change
- A test with two humans in it fails on the exact line that dropped the second.
  Proven red against the unpatched script, two failures, then green
- Naming the counter by the resolved label turns `owner:3` on a family member's session
  from a lie into a tell. A wrong name in a header is visible. A missing row is
  not

## Watch-outs

- The reviewer is also a reader built for the first principal. A session that has
  only ever seen one-sided transcripts has no expectation of the other side. The
  cheap check is to compare message counts in the header against lines printed,
  and to ask, for any conversation, where the other person's words are
- This is not the same as `unknown-value-renders-as-absence`. There the value is
  outside the renderer's vocabulary. Here the value is a known, valid label that
  the renderer was simply never taught to print
- Adding the second person to the parser and not the renderer is the same shape
  as `a-grant-the-doctrine-denies-is-reported-as-a-bug`: the mechanism moved and
  a sentence that assumed the old world stayed

## When to Use

Any time a system that had one user, one account, one tenant, or one agent gains
a second. Grep every reader for the first one's name as a literal. Transcript
tools, notification routers, dashboards, log summarisers, and cost reports are
the usual suspects, because they were written first and read most.

## Adjacent Patterns

- `count-the-source-not-the-survivors.md` for why the fixture could not catch it
- `unknown-value-renders-as-absence.md` for the neighbouring failure with a value
  the renderer has never seen
- `an-unnamed-blind-spot-reads-as-an-empty-source.md` for the same silence across
  hosts rather than inside one reader
- `the-lane-decides-who-approves-not-the-agent.md` for what those three invisible
  messages were asking for, and what was built for them the same day

## Source

The household-agent repo ebb02b4, 2026-09-07. State.db held three user rows for the
session, `read_messages.py --conversations-only` printed zero of them and a
header of `owner:3`. One `HUMAN_LABELS` set, a labelled header, a second person
in the fixture, and three new checks that fail on the old script
