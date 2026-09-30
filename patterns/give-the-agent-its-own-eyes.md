---
type: pattern
date: "2026-09-29"
source: The dashboard's art commission (a proof sheet the agent judges at actual size before installing) and a build session where 4 defects passed inspection and were caught by running the path
tags:
  - verification
  - agent-ui
  - visual
  - feedback-loop
---

# Give the Agent Its Own Eyes

Visual work with an agent used to go one way. The agent changes the thing, says "have a look", the person describes what is wrong, the agent changes it again. The person spends the afternoon as the agent's eyes, and every round costs a turn of their attention for a judgment the agent could have made itself if it could see.

The dashboard's art pieces (a landmark drawn beside a region of a member's world) went this way until the commission got a proof step. Now the flow is plan, score, run the check, look at the proof sheet it renders, judge it at the size it will actually be on the globe (about 120 to 170 pixels wide), correct at most twice, then install. The member sees the installed piece and reacts to that, not to the agent's guess about it.

The non-visual version is the same lesson. On 2026-07-18 a defect was recorded as fixed because a symlink existed. Running the gate showed it still dead: the gate resolved its probe as a filesystem sibling, and the thing it probed was itself a symlink. 4 defects came out of one session that way, each invisible to inspection and obvious to a run.

## The Pattern

Before an agent builds a thing, give it a way to see the result that does not go through a person:

- **A proof.** Render the output the way it will be seen (the proof sheet, a screenshot at actual size, the page in a headless browser) and have the agent look at it and say what it sees before it says it is done.
- **A saved case.** A link or a command that reproduces the exact state, so the agent tests against the case rather than against the person's description of it.
- **A checker with no person in the loop.** A command that answers "did it work" for this kind of item. A health probe beside a slow request, a timeline from a relay, a run of the actual path.
- **A written place to look.** The item names how it will be seen (the URL, the command, the proof) before anyone builds it. An item with no way to be seen is not ready to build.

And a correction budget: look, correct at most twice, then install and ask. The budget is what keeps the agent from polishing in private what the person has not yet reacted to.

## Why It Works

The person's reaction to the real thing is still the review. What changes is what they are reacting to. Without the agent's own eyes, they review the first draft, then the second, then the third, describing each. With them, they review the piece the agent already judged at actual size, and their attention goes to meaning and fit, which is the part only they can do.

It also corrects a specific overconfidence. An artifact's existence reads as done, and inspection cannot tell a live symlink from a dead one. Only the run can.

## When to Use

Any item whose result is seen rather than computed: layout, art, copy on a page, a voice line, a chart. And any fix that is being declared from the artifact rather than from the path it was meant to repair.

Near neighbours: [`health-check-that-never-exercises.md`](health-check-that-never-exercises.md) is a check that answers the wrong question. This is the agent having no check at all and borrowing the person's eyes instead. [`appending-is-not-learning.md`](appending-is-not-learning.md) is where the recurring mistake goes once the agent can see it: into a check, not a paragraph.

## Source

A member's second-brain dashboard: the region and tracker art commission, which renders a proof sheet the agent judges before installing, decided with a collaborator on 2026-09-26. The symlink case is from a personal agent's local-model work on 2026-07-18.
