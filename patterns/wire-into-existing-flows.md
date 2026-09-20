---
type: pattern
date: "2026-05-01"
source: A course participant's home-agent system (crows-nest dormancy at 28 days; system-understanding wired into the household agent's CLAUDE.md)
tags:
  - adoption
  - discipline
  - meta
---

# Wire Adoptions Into Existing Flows

When you adopt a pattern from elsewhere — a skill, a hook, a doc, a check, a pre-commit gate — the work isn't done when the file exists in the repo. The work is done when something fires it on a normal day. New artifacts that sit standalone go unused. The fix is to identify a forcing function and wire to it in the same commit that creates the artifact.

## The Problem

The dominant failure mode of pattern adoption is *standalone artifact*. You read a great pattern, you create the matching file in your project, you push, you move on. Three months later the file is unchanged, never invoked, possibly still correct in theory, completely useless in practice.

Concrete instance from a course participant's home-agent system: the **crows-nest** skill. The agent's account proposed it 2026-04-03; the participant merged same-day, no comments, no modifications. Twenty-eight days later: zero `_crows-nest/` output, zero git commits referencing crows-nest, zero modifications to the skill file. The skill was correct. Nothing fired it. The merge was frictionless because the cost was deferred — and the cost was that the skill was invisible to the participant's actual workflow.

Compare to the **humanizer** skill (also from the participant's system, also adopted by the author). Humanizer was wired into the existing voice-pass workflow as a second scrub. Every draft now triggers it because the voice-pass step calls it. Same kind of artifact, different fate, because of the wiring.

The pattern: **adoption is the wire, not the file.**

## The Pattern

Three steps, in order:

### 1. Find an existing forcing function

A forcing function is something that fires on a normal day without anyone deciding to invoke it. Examples in priority order:

- **A CLAUDE.md rule** that the active session reads at every turn (highest leverage in Claude Code projects)
- **A session-start checklist** that gets run at the top of every session
- **A scheduled task** (cron, launchd, GitHub Actions cron) that fires on its own
- **A pre-commit / pre-push hook** that fires on every commit or push
- **A skill that's already invoked daily** that you can extend with a sub-skill or reference
- **A template that gets used** (project README, daily note, standard PR description) that you can extend with a checklist item

The order matters: prefer the highest-leverage forcing function the artifact's nature allows. A new validation rule belongs in CLAUDE.md or a hook, not in a doc nobody re-reads.

### 2. Add the wiring edit in the same commit as the artifact

Don't ship the artifact in PR #1 and the wiring in PR #2. They're one change. Splitting them is how the wiring gets forgotten.

The wiring edit is usually small — one line in CLAUDE.md, one entry in a checklist, one row in a table. It looks underwhelming next to the artifact. Ship them together anyway. The wiring is what makes the artifact load-bearing.

### 3. Test on the next real case

The first invocation reveals the gaps. Names that drifted, paths that don't exist, commands that fail. Don't wait for an organic case to hit it — pick the next question or task that the artifact would naturally handle and run the artifact through it explicitly. The friction tells you what to fix.

## What to wire to (by artifact type)

| Artifact | Best-fit forcing function |
|---|---|
| Anti-fabrication / verification skill | A non-negotiable rule in the active CLAUDE.md ("cite atomically when stating system facts; use skills/.../system-understanding/SKILL.md") |
| Verification recipe in a living doc | The doc itself — but also referenced from any session-start checklist that points at the doc |
| Skill that produces output (report, summary, audit) | A scheduled task that runs it on a cadence + a downstream consumer that reads its output |
| Author-guidance pattern (TDD-for-skills, decision-doc-ADR) | A PR-description template that asks "did you do this?" + a template file in the repo so there's a path to start from |
| Code-quality rule (gotcha comments, framework-specific behavior) | A pre-commit hook if mechanical; a CLAUDE.md rule if judgment-shaped |
| Doc shape / convention (a planned subsystem, status-categories) | The session-start checklist that points at the doc + a one-line entry in the relevant template |

## Watch-outs

**The wiring goes stale, not the artifact.** As projects evolve, the forcing function may move (CLAUDE.md gets restructured, the checklist gets renamed, the cron gets retired). When that happens, the artifact looks fine but no longer fires. The audit question is: *"if I deleted the artifact today, what would break?"* If nothing would break, the wiring has decayed and needs to be redone.

**Wiring fatigue is a real risk.** If every adoption adds a CLAUDE.md rule, the file balloons and the rules stop being read. Two ways to manage: (a) keep wirings *bite-sized* (one line, not five), (b) batch related artifacts under a shared "artifact cluster" naming so they reinforce rather than compete (see "Four-artifacts-named-together" sub-pattern below).

**Don't fake the wiring.** A reference at the bottom of a deep doc is not wiring. The wiring has to be at a place that gets read on a normal day. Burying the reference is how the artifact still doesn't fire even though you "wired it."

## Sub-pattern: Four-artifacts-named-together

When adopting a pattern produces multiple related artifacts (tool + protocol + reader contract + author guidance), naming the relationships explicitly is part of the wiring. Otherwise future agents pick one and bypass the others.

The shape:

| Artifact | Role |
|---|---|
| Tool | Runs the actual checks. (e.g., `catchup.sh`) |
| Protocol | How to *cite* what the tool found. (e.g., `system-understanding` skill) |
| Reader contract | Tells the reader to verify before acting. (e.g., ROADMAP staleness preamble) |
| Author guidance + template | When *building* a state-grounded skill, follow this. (e.g., `tdd-for-skills` + baseline template) |

The cluster lives as a small table in the active CLAUDE.md or equivalent. The point isn't the table; it's that the relationships are visible. Picking one artifact and bypassing the others is the failure mode the table prevents.

## Examples to learn from

**Success: humanizer-second-pass.** Adopted into the author's vault by wiring into the existing voice-pass step. Every draft now fires it. The skill is correct *and* used. (See `humanizer-second-pass.md`.)

**Success: system-understanding in the household-agent repo (2026-05-01).** Skill at `skills/shared/system-understanding/`, plus a Non-negotiable CLAUDE.md rule that names it, plus a "State questions — the artifact cluster" section that names its relationship to catchup.sh, ROADMAP preamble, and the baselines template. Three wiring edits in the same commit as the skill creation.

**Failure: crows-nest.** Adopted into the participant's the workspace repo 2026-04-03. No wiring — it's invoked by Claude Code reading the SKILL.md, but nothing in his daily flow points at it. 28 days of zero invocations as of 2026-05-01; **71 days as of 2026-06-13 — still never run, still never modified, still zero output.** A clean PR that sounds useful merges with near-zero friction precisely because the cost is deferred; the deferred cost is permanent invisibility. **Merge ≠ adoption — run-rate is how you detect a non-adopted artifact** (the audit question "if I deleted this today, what would break?" is run-rate by another name). And a second reason crows-nest specifically rotted: it duplicated something the target already does himself — horizon-scanning via `roadmap-riff` + meta_system thoughts. Wiring can't save an artifact that competes with an existing instinct. Pitch what the target *can't or won't* do by hand, not what they already do.

**Failure mode to watch for: "two stale issues on archived repos."** the agent's account opened two issues on what later became archived the participant's system repos. The issues are on dead-letter surfaces — they exist, they're not dead-deleted, but nothing fires them and they decay quietly. Adopting a pattern into the wrong forcing function is the same shape: the artifact exists, it just won't fire.

## How to Adopt

1. When you're about to create a new artifact (skill, hook, doc, check), pause and ask: *what fires this on a normal day?*
2. If the answer is "someone has to remember to invoke it," that's a standalone-artifact red flag. Pick a forcing function instead.
3. Write both edits in the same commit — the artifact and the wiring.
4. After committing, find the next real case that the artifact would handle and run it explicitly. Note any friction. Fix.
5. Add an audit habit: every quarter (or per project cadence), ask of every adopted pattern: *"if I deleted this today, what would break?"* If nothing breaks, redo the wiring or remove the artifact.

## Adjacent Patterns

- **TDD-for-skills** (`tdd-for-skills.md`) — when you're shipping a skill, the wiring includes the baseline-behaviors RED phase. Don't wire untested artifacts into forcing functions.
- **Doc-with-warning-preamble** (`doc-warning-preamble.md`) — the preamble is itself a wiring artifact. It has to be read on a normal day to fire. Place it where readers actually look (top of the section, not bottom).
- **Decision-doc ADR** (`decision-doc-adr.md`) — decision docs that aren't referenced from CLAUDE.md or the relevant code's commit message decay into forgotten lore. Wire by referencing.
- **Personality-as-discipline** (`personality-as-discipline.md`) — the persona is itself a wiring mechanism. Discipline encoded as personality fires every turn the agent responds.
