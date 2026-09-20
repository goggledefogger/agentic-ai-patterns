---
type: pattern
date: "2026-08-30"
source: The personal agent (the author's chief-of-staff router), the agent's ops playbook rule 90 — the author's standing directive 2026-08-30, given while triaging a batch of voice notes after the triage lane filed to local docs and was about to post to a shared GitHub board.
tags:
  - permissions
  - agent-autonomy
  - unattended
  - approval
  - layered-defense
  - llm-policy
---

# A Tier Says What You May Touch, Not What Others May See

## The Problem

Agent permission systems converge on one axis: a sensitivity tier per resource (public/internal/private/shared-IP), gating what an agent may read and write. It's shaped entirely around leaks. But it silently answers a second question it was never designed for: once a tier clears a write, the agent writes, and if that write is a board comment, a PR review, a push, or a message, the agent has just published in the principal's name. The tier said yes to that exactly as readily as it said yes to a local file edit.

This inverts the tier's own intuition. The most open resources are the most exposed, not the least: an open repo has no gate at all, which makes it the easiest place to put something half-formed in front of colleagues. The harm isn't disclosure, so no confidentiality control ever fires. The harm is that a wrong or premature thing now exists where other people can see it, and a board comment can't be quietly re-edited the way a local file can.

The personal agent hit this on 2026-08-30, mid-triage of a batch of voice notes: the triage lane correctly filed several items to local docs, then queued the same kind of write toward a shared GitHub board. The tier had cleared both. Only one of them should have gone out without a stop.

## The Pattern

Two orthogonal axes, checked separately.

**Axis 1, sensitivity:** may this agent touch this store at all. Answered by tier plus deterministic gates. Existing art — see `sensitivity-tiered-access-control.md`.

**Axis 2, visibility:** does this action change what another person sees. Answered by the action, never by the resource.

Split every capability by axis 2 into free and draft-and-ask.

**Free, once the tier clears:**
- local file writes and edits (provided they are non-destructive)
- local commits (a commit on an unpushed branch is reachable by nobody)
- parking a draft in an inbox the principal reads

**Draft-and-ask, regardless of tier (Visibility / Impact):**
- posting or commenting on a shared board, issue, or PR
- push, or anything that makes a branch reachable by a collaborator
- deploying or merging code, as opposed to writing about the change
- any send — email, chat, a message to another agent
- a write into a store another person queries
- executing destructive local actions (e.g., deleting data or running risky scripts)

The deliverable for the second list is a draft plus a one-line ask, and crucially, the work still gets done in the same pass. This is not deferral. Produce the exact comment body, then hand it over. Deferring the thinking is the failure mode this pattern is most often mistaken for.

Reporting is part of the pattern. Report landed work and waiting work as two labeled lists. Folding a pending item into the completed list is the easy sentence to write, because both halves succeeded from the agent's side, and a principal told something is done stops watching for it.

## Watch-outs

- The axis is not derivable from the tier, and any implementation that stores it per-resource will drift. It's a property of the verb, not the noun
- "Unattended" is the trigger, not "autonomous." A human watching the turn can approve inline. A cron, a poller, a voice-triage lane, or a subagent finishing its own job can't ask, and that's precisely when the ceiling binds
- A local commit is free only while the branch stays unpushed. If the workflow auto-pushes, the commit is a publish and moves lists
- This supersedes any standing "file it the same turn, unprompted" reflex for outward surfaces. Reinterpret those as "draft it the same turn," not drop them, or the pattern eats a good habit along with the bad one
- No gate enforces this. It's an honor-system layer in the sense of `sensitivity-tiered-access-control.md`'s Layer 3, and it should say so plainly rather than imply a hook is watching

## Adjacent Patterns

- `sensitivity-tiered-access-control.md` — the one-axis ancestor. This pattern is the second axis it never modeled
- `approval-scope-invisible-to-gates.md` — a different axis split, identity versus scope, same shape: a gate answering the question it's built for while missing the one that matters
- `unattended-run-discipline.md` — the context this pattern's trigger depends on. No human at the wheel is what turns "may touch" into "may publish"
- `eyes-free-approval-channel.md` — how the ask in draft-and-ask reaches a human who isn't at a desk to see it
- `the-lane-decides-who-approves-not-the-agent.md` — the one shape of outward write that legitimately skips draft-and-ask: a closed verb list, one manifest line, reversible by the same lane
