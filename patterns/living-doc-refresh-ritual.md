---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo)
tags:
  - documentation
  - drift
  - governance
---

# Living-Doc Refresh Ritual

Split any doc that captures system state into stable, living, and distillate layers, then refresh the living layer by verifying against the actual running system — not against the last refresh.

## The Problem

Architecture docs drift within weeks. The usual fix ("update more often") doesn't work because there's no forcing function, and by the time the drift is obvious, the doc is untrusted anyway. The real fix isn't faster updates — it's contained drift plus external verification.

## The Pattern

Three layers, each with its own update cadence:

- **Stable layer** — the constitution. Principles, thesis, problem statement, architectural invariants. Updated only via an explicit amendment process. Review quarterly.
- **Living layer** — the current state snapshot. Built / in-flight / to-build, sub-project status, known limitations, principle-coverage audit. Updated monthly or on material change. Carries an explicit `> Last refreshed: YYYY-MM-DD` marker.
- **Distillate** — the dense detail. Full rationale, component maps, decision logs, operational detail. Updated as needed.

## The Refresh Ritual

When the living layer gets stale:

1. **Verify against the running system**, not against memory or the previous refresh. SSH in, run the actual scripts, read the logs, query the database.
2. **List the deltas explicitly** in the refresh commit message. "X was estimated 50%, is actually 100%" beats "refreshed §9."
3. **Expect corrections in both directions.** You'll find things you thought were half-done that shipped, and things you thought shipped that regressed.
4. **Re-date the `> Last refreshed` marker.**

## Anti-pattern: the marker that became a changelog

The `> Last refreshed:` marker must stay a *date* (plus, at most, one line of headline). Its job is to tell a reader how stale the living layer might be — nothing more.

The failure mode: instead of putting the deltas in the commit message (step 2 above), each refresh appends them *inline to the marker*. Over weeks the marker swells into a running changelog — thousands of tokens of "what changed when," sitting at the top of a doc that gets read or auto-loaded every session. Observed in production: a single `> Last refreshed:` line carrying two full days of narrative deltas, ~6K tokens, before the actual status content even started.

The deltas belong in `git log`, where they're already captured and don't cost a re-read. If you want a durable narrative of changes, that's a decision doc ([`decision-doc-adr.md`](decision-doc-adr.md)), not the marker. Keep the marker to: `> Last refreshed: YYYY-MM-DD — one-line headline of the current state.`

## Example

The participant's product brief at `_bmad-output/product-brief.md` separates:
- §1–8 stable (principles P0–P9, problem, architecture, thesis)
- §9–10 living (Built / In-flight / To-build, with explicit percentages)
- Distillate in `product-brief-distillate.md`

The 2026-04-16 refresh of §9 (commit `44adcea`) was done via Tailscale to the production Mac Mini. Three material corrections surfaced:

- Memory Bridge: 50% → **100%** (shipped, consumed by 7 skills)
- The operations persona coordinator: 40% → **80%** (three-phase boot live)
- Plow/OpenClaw: "pristine" → **installed and running**

All three corrections were in the participant's favor — he underestimated his own progress. That's common: without verification, authors tend to carry forward caution from the last known state.

## When to Use

- Any system large enough that nobody carries the full state in their head.
- Any doc that someone will actually read when onboarding or making decisions.
- Any project with >~5 sub-components in different stages of maturity.

## When NOT to Use

- Small solo projects where the doc and the code are maintained by the same person in the same week. The overhead isn't worth it.
- Docs that are inherently static (READMEs for stable libraries, tutorials that don't change).

## Adjacent Patterns

- Pairs well with a `ROADMAP.md` using status categories (see [`templates/roadmap.md`](../templates/roadmap.md)) — the ROADMAP can be the living layer itself, and the refresh ritual applies to it directly.
- Pairs with decision-doc pattern (see [`decision-doc-adr.md`](decision-doc-adr.md)) — living layer stays high-altitude, deep rationale lives in dated decision docs.
- Complemented by [`living-doc-archive-split.md`](living-doc-archive-split.md) — this ritual keeps the living layer *honest* (no drift); the archive-split keeps it *lean* (no bloat). Drift and bloat are different failures of the same docs, and an auto-loaded living doc needs both fixes. The marker anti-pattern above is the bloat problem showing up inside the refresh mechanism itself.
