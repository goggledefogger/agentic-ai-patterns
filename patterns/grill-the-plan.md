---
type: pattern
date: "2026-05-02"
source: Reddit u/PitifulRice6719, "Matt Pocock's skills repo + Hermes sub-agents for feature work" (r/hermesagent, 2026-04-29) — referencing Matt Pocock's `grill-me` and `grill-with-docs` skills at github.com/mattpocock/skills
tags:
  - planning
  - prd
  - bmad
  - hard-questions
---

# Grill the Plan

Insert a hard-questions step between draft-plan and PRD. Have an adversarial sub-skill (or a deliberately skeptical pass) interrogate the plan — scope, risks, surfaces touched, unknowns — and force the plan to answer before any code or PRD ceremony begins.

## The Problem

The default plan-to-PRD flow is too smooth. The user describes a feature, the agent writes a plan, the agent writes a PRD, the agent generates issues, agents start coding. By the time anyone notices the plan was vague — "we'll figure out auth later," "this assumes the existing rate limiter is fine" — there's already a PRD and a story list inheriting the same gaps.

The friction comes from the wrong direction: it shows up when implementation hits a wall, when the agent gets stuck, or when integration reveals the spec was incomplete. By then the cost of going back is the rework of everything downstream.

What's missing is a step *before* the PRD where the plan has to defend itself.

## The Pattern

Three artifacts, in order:

1. **Plan draft** — markdown file. Scope, risks, surfaces touched, rough outline. No code, no PRD ceremony yet.
2. **Grill pass** — a sub-skill or sub-agent runs against the plan with an adversarial brief: *find the unknowns, the unstated assumptions, the missing surfaces, the places "we'll figure out later" is doing load-bearing work*. Output is a list of hard questions and gaps.
3. **Plan revision** — author answers the grill output. Either by amending the plan with explicit decisions, or by accepting the gap and writing it down as an open question to surface in the PRD.

Only after the plan survives the grill does PRD generation start. The PRD then inherits an interrogated plan instead of a smooth-but-vague one.

The Reddit post's phrasing for the loop: *"Run grill-me / grill-with-docs on the plan. Answer the hard questions. If it drifts, tell it to refocus on the feature. Let it update the plan."* The drift-correction is part of the loop — adversarial passes will sometimes wander into adjacent concerns; the user keeps the grill scoped to the feature actually being shipped.

## What good grilling looks like

A useful grill pass produces questions of these shapes:

- **Surface omissions** — "the plan describes a backend route and a UI button. What about the empty state? The loading state? The error toast?"
- **Stated-but-undefined dependencies** — "the plan says 'uses the existing auth middleware.' Which one? `requireAuth` or `requireSession`? They behave differently for unauthenticated requests."
- **Implicit assumptions** — "the plan assumes the rate limiter is fine. The current rate limiter is in-memory per process. Is the new endpoint sticky-sessioned, or does it need a shared limiter?"
- **Missing failure modes** — "the plan has a happy path. What happens when the third-party API is down for the whole call?"
- **Scope creep candidates** — "the plan mentions 'while we're here, we should also...' three times. Which of those are actually in scope, and which are getting decided silently?"

What grilling should *not* do is propose redesigns. It surfaces gaps; the author closes them. Letting grill drift into "here's how to redesign this" defeats the purpose — you're back to a smooth flow that bypasses interrogation.

## Why a separate pass beats inline questioning

Two reasons.

**The default model voice is collaborative.** When the same model that drafts the plan is asked to critique it, the critique is mild — it preserves the plan's framing because the model wrote it. A separate pass with a deliberately adversarial brief breaks that frame. (Pocock's `grill-me` is one implementation; an explicit prompt like *"You are reviewing this plan adversarially. Find the gaps a senior engineer would catch in PR review."* is another.)

**The author needs a moment to defend.** Inline questioning tempts the author to keep editing. A separate pass forces the question list to crystallize before the editing begins, which is the only way to know whether each gap got an answer or got lost in revision.

## When to Use

- Multi-surface features where the cost of a wrong PRD is days of agent-hours
- Plans drafted by AI agents (which tend toward smooth, plausible coverage of stated requirements while missing unstated ones)
- Refactors that touch shared code — implicit assumptions are most expensive there
- BMAD flows where the cost of `bmad-create-prd` → `bmad-create-epics-and-stories` → execution is high enough that a hard pause before PRD pays off

## When NOT to Use

- Single-surface tweaks where the plan is "edit this file, change this thing" — there's nothing to grill
- Genuine experiments where the unknowns are the point — grilling an exploration produces noise
- Time-pressed hotfixes where the cost of grilling exceeds the cost of debugging the gap later

## Watch-outs

**Grilling can become a stall mechanism.** If every plan gets grilled into infinity, the loop is broken. Two questions cap it: (1) *did the plan answer most of the gaps?* — if yes, accept remaining gaps as known unknowns and move on. (2) *are the open questions things the PRD can answer, or things only implementation will answer?* — if implementation, mark them and move on; the grill isn't going to resolve them.

**Drift-correction is real work.** Adversarial passes wander; the user has to pull them back to the feature actually being shipped. The Reddit post mentions this explicitly: *"if it drifts, tell it to refocus on the feature."* Plan for the drift; it's not a sign the pattern is broken.

**Don't conflate grilling with PRD validation.** The PRD validation pass (BMAD's `bmad-validate-prd`, or its equivalent) checks that the PRD is complete and well-formed. Grilling checks that the plan was honest. They're at different layers — grill the plan first, validate the PRD second.

## Adjacent Patterns

- `vertical-issue-split.md` — once the plan survives grilling, decompose vertically. A grilled-but-horizontal split still ends in integration drift.
- `tdd-for-skills.md` — RED-phase baselines are a different shape of the same idea: interrogate before you implement. The two patterns reinforce each other; both fight smooth-but-fabricated work.
- BMAD's `bmad-validate-prd` and `bmad-check-implementation-readiness` — downstream gates that operate on the PRD and story list. Grilling complements them by hardening what comes in.
- `wire-into-existing-flows.md` — grilling needs to be a step in your plan-to-PRD flow, not a standalone doc that gets read once. Wire it as a skill, an LLM sub-call, or a checklist entry in the PRD-creation template.
