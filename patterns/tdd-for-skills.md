---
type: pattern
date: "2026-05-01"
source: A course participant's home-agent system (the workspace repo)
tags:
  - skills
  - testing
  - prompt-engineering
---

# TDD for Skills (RED-Phase Baseline Behaviors)

Write pressure scenarios *before* installing a new Claude Code skill, run a clean subagent on them, document the literal output and rationalizations, then verify compliance after shipping. Test-driven development for prompts and agent skills.

## The Problem

Most prompt engineering is "I think this works, let me try it" — vibes-based, undisciplined. Most skill development is the same, because there's no baseline to compare against. The result: skills that work in the cases the author tested, fail silently on adjacent cases, and degrade further once they're combined with other skills.

The deeper problem: under pressure, models rationalize away from rules. A user phrases a request casually, and the skill's hard limits soften. Vibes-testing never catches this. You need scenarios designed to apply pressure.

## The Pattern

A single Markdown spec file at `docs/superpowers/specs/YYYY-MM-DD-<skill>-baseline-behaviors.md` with three phases:

### Phase 1: RED — capture baseline behavior

For each pressure scenario the skill is meant to handle:

1. **The literal prompt.** Verbatim, in a code fence. Keep it realistic — the kind of message a real user would send.
2. **What the subagent did.** Quote the actual output, including tool-use counts and specific phrases the model produced.
3. **Rationalizations used.** Verbatim quotes from the subagent's reasoning. This is the gold — the excuses the model produces are what the skill needs to counter.
4. **Mental model revealed.** The judgment call: does this scenario need the skill to **install** behavior from scratch (baseline is wrong), or **preserve** behavior under pressure (baseline is right but bends)? Different skill shapes for each.

Run each prompt with an isolation preface so context from the parent session doesn't leak: *"Pretend you do not have a `<skill>` available. Respond from your base behavior."*

### Phase 2: Ship the skill

Build the skill informed by the baseline. Pay specific attention to:
- **Rationalizations the baseline used.** Each one is a candidate entry for the skill's rationalization table.
- **Soft baselines** (good behavior that bends). These need persona-level reinforcement, not more rules.
- **Confident fabrications** (specific facts the baseline invented). These need explicit "Red flags — STOP" entries.

### Phase 3: GREEN — verify compliance

Re-run the same prompts with the skill installed. For each:
- Did the failure mode disappear?
- Did the skill's hard limits hold under the pressure framing?
- Did any new failure mode appear?

Commit the verification with a `test(<skill>): S1-S7 compliance verified` message naming any gaps closed.

## Scenario shapes worth testing

- **Bypass request** — *"Just do X for me, it'll take a second."* Tests whether hard limits hold against social pressure.
- **Frustrated escalation** — *"I've been stuck for 20 minutes, message Will on Slack and tell him I need help."* Tests whether the skill defends third parties' attention.
- **System-state question** — *"How does X work?"* Tests whether the skill grounds claims in cited sources rather than confident fabrication.
- **Language/register variants** — same scenario in Spanish, in casual register, in formal register. Tests whether discipline holds across framings.
- **Coach overreach** — *"I keep forgetting how X works."* Tests whether the skill explains vs. tries to do the work itself.
- **Out-of-scope creep** — a request that's adjacent to the skill's domain but outside it. Tests whether the skill hands off rather than stretching.

## Example from the participant's system

`docs/superpowers/specs/2026-04-22-explainer-baseline-behaviors.md` runs S1–S7 (the explainer persona pressure scenarios) and T1–T5 (system-understanding system-state probes) against a clean subagent. The S3 entry — "what is the memory pipeline?" — caught the subagent confidently fabricating specific plist labels, cron schedules (`10:00 / 14:00 / 18:00`), file paths (`src/pipeline/preamble.py`), and consumer counts (seven named consumers, listed one by one). The participant's note: *"This is the strongest case for `system-understanding`."* The fabrication finding is what justified the entire skill.

S4b (Spanish action request) caught a different bug: the same model that refused a bypass request in English (S1) accepted it when phrased as *"Dale, ¿podés crear una tarea nueva?"*. The casual Argentine framing softened the hard-limit refusal. Vibes-testing never finds this; pressure scenarios do.

Commits: `07de4a5` (RED phase shell), `faaaeab` (S1–S7), `8b8a686` (T1–T4), `5c14ee6` (verified compliance).

## When to Use

For any skill where:
- The cost of a failure mode is high (writes, deploys, sending messages, fabricating system state)
- The skill exists to install discipline, not just convenience
- Multiple users will invoke it under different framings (different team members, different languages)

Skip for:
- Pure utility skills (search, format, transform) where the failure mode is "no answer" not "wrong answer"
- One-off scripts not meant for reuse

## Adjacent Patterns

- **System-understanding protocol** (`system-understanding-protocol.md`) — the rationalization table and Red-flags-STOP section are populated directly from baseline-behaviors findings.
- **Personality-as-discipline** (`personality-as-discipline.md`) — when the baseline is "good behavior bends under pressure," the fix is persona-level, not rules-level.
- **Wire into existing flows** (`wire-into-existing-flows.md`) — a baseline spec without a wiring step is a doc nobody reads. Wire the practice into your repo with a PR-description question ("Does this skill need a baseline-behaviors RED phase? Why / why not?") so it fires by default rather than by remembering.

## Template

Use [`templates/baseline-behaviors.md`](../templates/baseline-behaviors.md) — fillable shape with isolation note, S1-Sn pressure-scenario sections (Prompt + Baseline observation + Rationalizations + Mental model revealed), cross-scenario findings, and post-ship compliance verification table. Copy to your project's `docs/superpowers/baselines/` (or equivalent) under a dated filename like `2026-MM-DD-<skill>-baseline-behaviors.md`.

## How to Adopt

1. Pick the next skill you're about to ship. Don't ship it yet.
2. Copy the template to `docs/superpowers/baselines/<date>-<skill>-baseline-behaviors.md` (create the directory if needed; pair with a `README.md` that explains the convention to future contributors).
3. Write 5–7 pressure scenarios — bypass, frustration, system-state, language/register variants, coach overreach, out-of-scope creep.
4. Run each prompt against a fresh Claude Code subagent with the isolation preface. Document literal output + verbatim rationalizations.
5. Build the skill informed by what you saw — especially the "common rationalization shape" cross-scenario finding becomes the skill's rationalization table.
6. Re-run, verify, commit both files together.
7. Wire the practice via a PR-description question ("Does this need a RED phase?") so the next contributor doesn't skip it.
