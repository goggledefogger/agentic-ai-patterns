---
type: pattern
date: "2026-06-10"
source: The work vault — design + UX pass on a D3 visualization tool, 2026-06-10. Validated in one full cycle: ~32 findings from two parallel reviewers, all implemented by a single subagent, verified state-by-state in a browser, followed by a pixel-diff-gated refactor. Generic guidance extracted into a skill's viz reference the same day.
tags:
  - subagents
  - workflow
  - design-review
  - frontend
  - verification
---

# Parallel Review → Merged Spec → Implementer → Pixel-Diff Gate

A four-stage shape for polishing a working UI with subagents: capture baselines, fan out independent reviewers, merge their findings into one conflict-resolved spec for a single implementer, then verify against the baselines. Refactors that should not change rendering get their own pass behind a pixel-diff gate.

## The Problem

Asking one agent to "improve the design" of a working UI conflates four jobs with different failure modes:

- **Critique needs independent eyes.** A styling lens (typography, color, contrast, atmosphere) and a UX lens (affordances, feedback, state coherence, safety defaults) find different bugs in the same screen. One generalist pass blends them and shallows both.
- **Critique and implementation contaminate each other.** A reviewer who will also implement starts pre-filtering findings by implementation effort. Read-only reviewers don't.
- **Two reviewers disagree.** Both may propose fixes for the same root cause with incompatible mechanics. Letting the implementer resolve that mid-edit produces hybrids that do neither.
- **Refactors hide regressions inside churn.** A tokenization or cleanup pass touching 100+ lines "should" be invisible on screen; without a hard check, drift slips through.

## The Pattern

1. **Baseline first.** Screenshot every distinct UI state (each tab, each mode toggle, a deep-interaction state, a narrow viewport) before anyone touches code. The baselines are both the reviewers' evidence and the verifier's reference.
2. **Fan out read-only reviewers in parallel.** Each gets the source, the full screenshot set, an explicit lens, and known pain points to investigate rather than rediscover. Require per-finding severity, root cause with line numbers, and a fix concrete enough to implement verbatim. Read-only is a hard rule.
3. **Merge into one spec, resolving conflicts yourself.** The controller (not the implementer) picks between competing fixes, sequences dependent items, and states the resolution inline ("two reviews gave different fixes for X, use Y"). The implementer gets one authoritative document.
4. **One implementer, then controller verification.** A single subagent applies the whole spec (no browser, syntax-check only); the controller re-walks every baseline state live and fixes or sends back what broke. Small one-line breakages are cheaper to fix in the controller's hands than to round-trip.
5. **Refactor passes get a pixel-diff gate.** Anything claiming zero visual change (design-token extraction, constant consolidation) runs as its own pass and must prove it: render the same state, diff against the post-design-pass screenshot, accept only sub-noise deltas.

## Why the stages stay separate

The merge step is the load-bearing one. In the source run, both reviewers demanded a fix for the same label-contrast collapse with different mechanics (HCL-luminance switch vs HSL threshold plus stroke scaling); the merged spec picked one and folded the other's stroke-scaling detail in. An implementer handed two raw reports would have had to make that call mid-edit, with the least context.

The pixel-diff gate caught its value immediately: the refactor was 172 changed lines and the diff showed zero pixels above noise threshold, which converted "should be identical" into "is identical" for the cost of one screenshot.

## When not to use

- The UI isn't working yet — this polishes, it doesn't build. Critique of a half-built thing reviews scaffolding.
- The change is small enough that one person holds both lenses (a button style, one view). Ceremony scales with surface area, same logic as [subagent-ceremony-by-task-type](subagent-ceremony-by-task-type.md).
- No way to capture baselines (no headless browser, no stable states) — the verification half of the pattern is what makes the fan-out safe.
