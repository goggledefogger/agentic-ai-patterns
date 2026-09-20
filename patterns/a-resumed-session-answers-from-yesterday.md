---
type: pattern
date: "2026-08-18"
source: The dashboard member-zero audit (what-changed answered from a resumed session's stale context)
tags:
  - drift
  - honesty
  - sessions
  - anti-pattern
---

# A Resumed Session Answers From Yesterday

A long-lived or resumed session, asked a recurring question ("what's unsaved?",
"what's the status?"), will anchor on its own previous answer and patch it with
plausible deltas instead of re-running the ground-truth commands. Caught live:
asked what was unsaved, a resumed session repeated yesterday's thirty-three-item
inventory — against a real list of three — and invented updates for the
difference: a screenshot that never existed, an edit that wasn't there, "a peer
session took it." The fabricated deltas are the dangerous part: they read as
diligence. "Two things moved since you last asked" sounds like a fresh look; it
was a story consistent with an old memory.

The same session carries stale *instructions*: a skill file read at turn one is
the version it follows at turn forty, missing every rule added since.

## The Pattern

Staleness leaks through every layer, so freshness must be demanded at every
layer it leaks through:

1. **The skill text**: the recurring question's instructions open with "run
   this fresh, every single time — your own earlier answer in this
   conversation is not a source; if memory disagrees with the commands, the
   commands win. Never patch an old answer with guessed deltas."
2. **The standing prompt**: don't say "read and follow X"; say "read X *again*
   each time — it changes between asks." A session that already knows a file
   will not reopen it unasked.
3. **The asking surface**: when a button or shortcut types the recurring
   question, the phrase itself carries the demand — "…check fresh right now,
   don't reuse an earlier answer." It steers the agent and teaches the human
   what freshness means, in one utterance.
4. **The audit**: an answer to a recurring question is checkable — diff it
   against the ground truth it claims to summarize. This failure was caught
   only because a second party compared the answer to disk.

## Why It Matters

Resume is worth keeping — continuity is real value ("say the word and I'll
save"). The fix is not fresh sessions everywhere; it is refusing to let
continuity of *conversation* become continuity of *facts*. Related:
`stale-pointer-asserts-confidently` (a stale fact copied elsewhere asserts),
`freshness-axis-must-match-the-question` — this is the conversational organ of
the same disease.
