---
type: pattern
date: "2026-05-02"
source: Reddit u/PitifulRice6719, "Matt Pocock's skills repo + Hermes sub-agents for feature work" (r/hermesagent, 2026-04-29) — referencing Matt Pocock's `to-issues` skill at github.com/mattpocock/skills
tags:
  - planning
  - decomposition
  - sub-agents
  - bmad
---

# Vertical Issue Split

When you decompose a feature into issues for AI agents to pick up, split *vertically* — each issue covers a thin slice from user-visible flow down through backend — not *horizontally* by layer (one issue per layer). Vertical slices integrate. Horizontal layers don't.

## The Problem

The default decomposition agents reach for is by layer, because layers are how engineers organize their mental model:

```
Issue 1: Backend — add the API endpoint
Issue 2: Frontend — build the UI for the new flow
Issue 3: Docs — write the user-facing doc
Issue 4: Tests — add coverage
```

This looks tidy. It is the wrong shape for both human review and agent dispatch.

When you run an agent against Issue 1, it ships a backend endpoint with no consumer, no error states, no demonstration that the contract makes sense. Issue 2 picks up days later, discovers the contract was wrong, the agent invents adjustments, the integration drifts. Issue 3 writes docs against a pair-of-pieces that don't behave together yet. Issue 4 writes tests against a system that's still being assembled.

The integration risk all collects at the end, when the slowest, hardest feedback loop kicks in: trying to use the feature end-to-end and discovering the seams don't fit.

## The Pattern

Split by user-visible slice, not by layer. Each issue is a thin vertical that includes whatever backend, frontend, glue, and tests are needed to make a single coherent piece of behavior real. Bad vs. good:

**Bad (horizontal):**
```
1. backend
2. frontend
3. docs
4. tests
```

**Good (vertical):**
```
1. happy-path slice — minimal backend route + the one frontend entry point that hits it + a smoke test that proves they connect
2. user-facing flow — the full happy-path UX with one error state + the backend behavior for that error
3. docs/landing update — describe the behavior that actually exists after slice 2
4. verification, additional error coverage, cleanup
```

Each slice integrates. Reviewing slice 1 means clicking the entry point and watching the backend respond — not reading a PR diff and trusting that some future slice will exercise the endpoint.

## Why this matters more for AI agents than for humans

A human running on a horizontal split notices the contract problem on day three when they pull the integration branch. The cost is one bad afternoon.

An AI agent running on a horizontal split *won't notice*. It'll fabricate plausible adjustments to make Issue 2's frontend match Issue 1's backend, or vice versa. The drift is invisible until the user tries the feature, by which point the agent has shipped four PRs against four issues and the diff is too tangled to debug.

Vertical splits force integration into the smallest possible loop. The agent can't ship slice 1 without proving the slice works end-to-end, because the slice is end-to-end by definition.

## The Verticalization Check

Before dispatching agents against a generated issue list, scan for layer names in titles. If you see "backend" / "frontend" / "API" / "UI" / "tests" as the only differentiator between issues, the split is horizontal. Reject and re-split.

A useful prompt at this step: *"Apply more verticalization. Each issue should be a thin user-visible slice that integrates from UI through API. Tests and docs accompany the slice they verify, not as standalone issues."*

The original Reddit post phrases it: *"Ask Hermes to apply more verticalization; it will know what you mean."* That's not magic — modern coding agents have read enough decomposition guidance to handle the term. The discipline is on the human to scan and reject the horizontal split *before* any agent starts implementing.

## Sub-pattern: HITL halt points

Vertical splits create natural human-in-the-loop checkpoints — typically at the end of each slice, where the integrated behavior can be reviewed in a browser or terminal in 60 seconds. Build the dispatch instructions so agents *halt* at these points instead of chaining into the next slice.

The Reddit post's phrasing: *"Implement the feature split slices … use sub-agents for every issue, parallelize when possible but carefully. Halt on tasks that are HITL or otherwise require human input."*

This works because vertical slices have natural review surfaces. Horizontal layers don't — there's no good "is this right" check between "the backend endpoint exists" and "the frontend exists" — both look done in isolation.

## When to Use

- Multi-surface feature work (backend + frontend + docs + tests, or any combination of >1 surface)
- Work that will be implemented by AI agents (Claude Code, Hermes sub-agents, Sandcastle, Codex, etc.)
- Refactors that touch >1 module — splitting by module is a horizontal smell; split by behavior preserved
- Any project using BMAD's epic/story decomposition where the stories risk decaying into per-layer chores

## When NOT to Use

- Single-surface tasks (tweak one CSS file, fix one regex) — there's no vertical to split
- Genuine infrastructure-only work (add a new database, swap a build tool) where there is no user-visible slice yet — split by deployable units instead
- One-person, one-session, one-day work where the horizontal/vertical distinction doesn't have time to bite

## Watch-outs

**Verticalization can over-rotate into "every slice ships a microservice."** Each slice should be the *thinnest* viable end-to-end change, not a self-contained product. If slice 1 needs an entire authentication layer to demo the happy path, the feature wasn't ready to decompose — go back to planning.

**Dependency tangles between slices need to be flagged in the issue list.** Slice 2 sometimes genuinely depends on slice 1's backend route existing. That's fine. The issue list should make the dependency explicit so agents don't try to parallelize work that isn't parallelizable. The Reddit post's *"parallelize when possible but carefully"* is doing the work of that warning.

**Don't replace this with one mega-issue.** "Build the whole feature" is also wrong — it's an un-split, which is worse than a horizontal split because there's no review surface at all. Vertical slices exist to create review surfaces; if you collapse them away, you lose the integration loop you were trying to enforce.

## Adjacent Patterns

- `wire-into-existing-flows.md` — the verticalization check itself needs a forcing function. Add it as a step in your plan-to-issues skill, not as a standalone doc.
- `tdd-for-skills.md` — for the agents *implementing* slices, RED-phase baselines protect against fabricated-integration shortcuts within a slice.
- BMAD `bmad-create-epics-and-stories` skill — when present, this pattern adds a verticalization scan to the existing flow rather than replacing it.
