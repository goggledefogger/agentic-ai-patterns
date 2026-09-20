---
type: pattern
date: "2026-05-02"
source: A course content engine vault — first-proof execution session 2026-05-02. Validated empirically across a 23-task plan split between mechanical-config tasks (Phase 1) and content-judgment tasks (Phase 3). ADR at `a doc`.
tags:
  - subagents
  - workflow
  - claude-code
  - discipline
  - cost
---

# Subagent Ceremony by Task Type

`superpowers:subagent-driven-development` prescribes a strict ceremony per task: implementer subagent → spec compliance reviewer subagent → code quality reviewer subagent → mark complete. That's three dispatches per task. Empirically, mechanical config tasks consume the reviewer dispatches and return "no issues found." Content/judgment tasks consume them and reliably catch real bugs. Split the ceremony by task type instead of running it uniformly.

## The Problem

Strict ceremony has fixed cost per task and value proportional to *cost-of-hidden-bug × difficulty-of-self-review*. The ratio collapses to zero on mechanical work and is high on content work. Running the same ceremony on both wastes ~30k tokens per dispatch on the mechanical side without catching anything. Skipping it on the content side propagates judgment errors downstream.

Concrete data points from a 23-task plan:

- **Task 1 (`.gitignore` append)** — full ceremony caught a real bug: a blanket `.claude/` rule was about to swallow `.claude/hooks/` and `.claude/settings.json`. High value. Three dispatches well spent.
- **Task 2 (BMad persona TOML, mechanical override)** — full ceremony returned "exact match, no issues found" twice. Zero actionable value, two dispatches wasted, ~30k tokens burned.
- **Tasks 3–12 (similar mechanical config: hook scripts, frontmatter additions, settings wiring)** — same shape. Reviewer dispatches consistently came back clean.
- **Phase 3 content tasks (extraction sheet, Blueprint slot synthesis, article draft, voice/humanizer pass)** — full ceremony caught: misattribution (the co-instructor vs. the author on a community-longevity claim), premature peer-slot promotion in the Blueprint (planning-first competing with the same anchor as the PRD article), em-dash voice violations, multiple judgment improvements. Skipping any of these would have shipped cracks.

The split is *not* "experienced devs skip ceremony." It's "the task itself signals which discipline is load-bearing."

## The Pattern

### Mechanical config tasks → implementer-only

Drop the spec and code-quality reviewer dispatches. Keep:

- **Fresh implementer per task** (context isolation still matters; this is the cheap protection that always pays).
- **Implementer self-review checklist** in the prompt template (the implementer is told what to verify before reporting back).
- **Controller spot-check on the report** — read the implementer's claims, verify file-exists / commit-landed via `Bash` `ls`/`git log`/`git show`. If anything is off, dispatch *one focused fix* instead of rolling out full ceremony retroactively.

Examples that fit this row:

- `.gitignore` edits, `.claude/settings.json` wiring
- Hook scripts that ship with explicit smoke-tests
- BMad persona TOML files (override scaffolds)
- Frontmatter additions to existing templates
- Single-file commits of installed-but-untracked artifacts (skill installs, snippet drops)

Test: *if I deleted the spec-reviewer dispatch on this task, would the bug it would catch be visible to a controller spot-check on the report?* If yes, implementer-only is correct.

### Content / judgment tasks → full ceremony

Keep the prescribed sequence:

- Implementer
- Spec compliance reviewer
- Code or content quality reviewer
- Fix-loop until both reviewers approve, *then* mark complete

Examples that fit this row:

- Extraction-sheet population (10-section template against a transcript)
- Blueprint synthesis (slot definition + corpus coverage)
- Article drafts (pillar or supporting, prose with source quotes)
- Voice + humanizer passes that touch prose
- Anything requiring source-quote accuracy, attribution correctness, or PM judgment

Test: *would a controller-eye spot-check on the report miss this bug?* If yes, full ceremony is correct. Quality issues here are usually invisible from outside the work — only the reviewer subagent reading the artifact end-to-end catches them.

### When in doubt, start mechanical and escalate

Adding ceremony costs less than discovering a shaky foundation downstream. If the implementer's report shows judgment calls or non-trivial integration, escalate that one task to full ceremony. Don't roll the upgrade out across the whole phase — keep it task-by-task.

## Why It Works

- **The split tracks the actual cost-value curve** instead of paying the same fixed cost on every task.
- **Spot-check + escalate beats blanket ceremony** because mechanical work usually gets caught at the spot-check anyway, and content work gets the discipline that earns its keep.
- **Phase-level rhythm emerges naturally**: Phase 1 plans tend to be mechanical (scaffolding, config), Phase 3 plans tend to be content (synthesis, prose). The split aligns with how plans are usually phased.

## Empirical results

From the validating run (23 tasks, mixed phases):

| Task type | Tasks | Strict ceremony dispatches | This-pattern dispatches | Reduction |
|---|---|---|---|---|
| Mechanical (Phase 1) | 12 | 36 | 12 + ~3 fix = 15 | ~58% |
| Content (Phase 3) | 5 | 15 | 15 (full ceremony kept) | 0% |
| Total | 23 | ~69 | ~45 | ~35% |

Quality observations: zero downstream issues traced to skipped ceremony on mechanical tasks. Multiple real bugs caught by reviewer dispatches on content tasks (listed in The Problem above).

## Watch-outs

**The borderline tasks are the trap.** A single-file edit to a Reference doc is mechanical-shaped *and* content-quality-meaningful. Default to mechanical, escalate if the spot-check finds a judgment issue. Don't try to nail the borderline — let the spot-check be the trigger.

**"It's just a small content task, I'll skip ceremony" is the failure mode on the content side.** Skipping reviewer dispatches on a 200-word article draft costs about as much as running them. The point of full ceremony on content is the *catch rate*, not the dispatch count. Don't rationalize.

**Code-quality review for content** doesn't have a dedicated `superpowers:content-reviewer` agent at time of writing. Use a `general-purpose` agent with a content-shaped review prompt. If a content reviewer ever ships, swap.

## Companion: route the *model*, not just the ceremony

The same task-type signal that decides *how much ceremony* can decide *which model*. Observed in a course participant's home-agent system (2026-06-13): across one window's commits, generative feature work stayed on the strongest model (Opus), while the meticulous, rule-heavy governance/cleanup work — a doc truth-sync, a churn-free-rendering fix, a single-source-config consolidation, an unattended-run-discipline pass — was co-authored by a newer, faster model (Fable 5). The split mirrors this pattern's logic: mechanical/rule-bound work tolerates (and benefits from the speed of) a lighter-ceremony, lighter-model path; open-ended judgment work earns the heavier model. Decide model tier off the same `mechanical` vs `content`/`judgment` label you're already applying. (Caveat: route by *empirical* fit, not by a fixed table — a "mechanical" task with a sharp correctness edge may still want the strong model.)

## How to Adopt

1. Encode the split in the project's CLAUDE.md (per `wire-into-existing-flows.md`). One short paragraph: mechanical → implementer-only, content → full ceremony, default mechanical with spot-check escalation.
2. When a plan lands, label each task `mechanical` or `content` at write-time. The label drives ceremony selection at execution time.
3. After execution, audit: did any mechanical-labeled task have a real bug caught only by the spot-check (i.e., would have shipped under strict implementer-only)? If yes, that's a label miscalibration — adjust the rubric, not the ceremony.
4. Promote to global `~/.claude/CLAUDE.md` if the split holds across multiple projects. Likely; the cost-value asymmetry is generic.

## Adjacent Patterns

- **wire-into-existing-flows** — the split lives as a CLAUDE.md rule, the only forcing function read every turn.
- **tdd-for-skills** — content tasks that involve shipping a *skill* should still ship with RED-phase baseline behaviors. Ceremony selection and TDD discipline are orthogonal.
- **vertical-issue-split** — pairs cleanly: vertical slices tend to mix mechanical (config, plumbing) and content (the actual feature behavior) within one slice. The slice-level ceremony plan benefits from labeling tasks as mechanical or content up front.
