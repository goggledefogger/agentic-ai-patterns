---
type: pattern
date: "2026-09-29"
source: The dashboard's handoff contract, measured on a 490-turn build session and refined over a month of real handoffs
tags:
  - handoff
  - context-engineering
  - cost
  - multi-session
  - long-running-agents
---

# Put a Thing Where It Is Read at the Frequency It Changes

Every turn re-sends the whole conversation, so a session's cost is roughly turns times average context, and average context only goes up. On one real build session it went from about 131k tokens per turn over the first hundred turns to about 579k over the last hundred, 4.4 times the price for the same kind of work. The member is right to start a fresh chat at a clean boundary. The handoff is what makes that cheap, and a bad handoff costs more than the context it replaced.

The bad handoff has one shape: three different things collapsed into one document, so the thing that changes every item drags the thing that never changes along with it, and the next agent reads all of it on its first turn.

## The Pattern

Split the handoff by how often each part changes, and read each part at that frequency.

| File | Holds | Changes | Read |
|---|---|---|---|
| Standing orders | Doctrine: invariants, verification discipline, traps, how the person works | Rarely | Once per session |
| Tracker | Position and history: the boxes, findings, a dated progress log, open decisions | Every item | The current box and its neighbours |
| Continuation prompt | Where we are, what is next, what the trap in the next item is | Every item | First, and it is short |

The prompt repeats neither of the other two. When it starts restating the doctrine, the agent receives every rule twice and the prompt grows forever.

A continuation prompt that rewrites itself is a ratchet, because the agent doing the rewrite is at the end of a long session, where keeping a rule feels free and dropping one feels risky. Four rules release it:

1. Before adding anything, delete: anything a test now enforces, anything true only of a closed item, anything the standing orders already say.
2. The reading list is named and scoped. Specific sections, specific boxes, never a whole document. A 5-document reading list can cost 50,000 words before the first line of code.
3. Protect the item briefing. The paragraph naming the next item, what already exists for it and the trap inside it is the only part no other document can regenerate. Cut everything else first.
4. State, not doctrine. Where the build is, what is next, how to run it, what to look at.

One thing the cuts must not eat: the standing orders open with the point of the whole build, quoted rather than summarised. An agent that knows what to build and not what it is for will satisfy the letter of a definition of done and will not know when to stop and ask. The test on any handoff is whether the agent can say, in one sentence and without opening another file, what this build is for and who is worse off if it is wrong.

## Why It Works

Each file is read by something that needs it at that cadence and by nothing else. The doctrine is read once, so it can be long. The tracker is read at the current box, so its history is free. The prompt is read first every time, so it is the only file whose length costs anything, and it is the only one with a delete-first rule.

A word ceiling on the prompt does not do this job. One was tried (400 words) and retired after a month, because a number policed a prompt that a reader should have been judging, and the four rules above were what had done the actual work.

## When to Use

Any agent work that outlives one session: a build tracker, a research thread, a migration. The tell that it is needed is a continuation prompt that is longer after ten items than after two, or a next session that opens by reading five documents top to bottom.

Near neighbours: [`the-conversation-is-unswept-state.md`](the-conversation-is-unswept-state.md) is about what the chat holds that nothing else does, this is about where each kind of thing should live once it leaves the chat. [`always-loaded-context-budget.md`](always-loaded-context-budget.md) is the same cost argument for files a harness loads on every turn.

## Source

A member's second-brain dashboard, whose Claude Code build sessions hand themselves to the next chat. The per-turn token measurement came from one 490-turn session's transcript. The 400-word ceiling was retired on 2026-09-11.
