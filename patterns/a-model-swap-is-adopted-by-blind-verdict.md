---
type: pattern
date: "2026-08-20"
source: The personal agent property-research lane — the one-property cheaper-model trial, judged same day
tags:
  - agents
  - model-routing
  - evaluation
  - reliability
---

# A Model Swap Is Adopted by Blind Verdict, Not by Vibes

Swapping a pipeline stage to a cheaper model is a change to *output quality* wearing the costume of a config change. The cheaper output will look right — same format, same section headings, passes the same mechanical gates — while being worse in exactly the dimension the gates cannot see. So the swap is an experiment with a verdict, and three rules make the verdict mean something: the judge is blind, the judge is never the producer, and **publication waits for the verdict**.

## The Problem

The personal agent, 2026-08-20. A property-research pipeline (deep county-records work: read the zoning code, run hazard queries with controls, pull permits) had cost $8–12 per house on the frontier model. One house was re-run with the research stage on a cheaper model to test a two-thirds cost cut.

The cheaper run finished in 8 minutes instead of 30–40, produced a 21KB fact pack instead of 100–130KB — and **passed every mechanical gate**, because the gates check topic *presence*, not depth (each required topic's vocabulary appeared). The pipeline published it.

A blind judge scoring both packs against the same 11-topic procedure returned 70% vs 100% — and the disqualifier was not thinness. The cheap pack had keyword-searched the zoning code for "short-term rental", found nothing, and concluded STRs were absent from the code — when the code's use table explicitly permits them with a conditional use permit. A buyer acting on it would have reached the wrong conclusion. It also stated hazard results without running the controls that prove the query works, and had comparable-sales data in hand without reading what it said.

The judge's classification mattered most: these were **diligence failures, not method failures**. The procedure was fine; the cheaper model didn't finish executing it. Diligence is precisely what erodes first when you shrink the model, and precisely what format-shaped gates cannot detect.

## The Pattern

1. **One swap, one house, one round.** Change the model for one unit of real work, alongside unchanged production runs of the same procedure — those are your comparison set, already paid for.
2. **Judge blind, against the procedure.** A separate judging context scores both outputs topic-by-topic against the written procedure, without being told which model produced which (or even that models differ). Ask for: scores, the weaker output's concrete gaps, whether gaps are diligence or method, and one binary — *publishable for the real decision, yes or no*.
3. **The judge is never the producer.** A model reviewing its own output ratifies it (`green-tests-can-mirror-the-same-guess` — a verifier handed its own answer verifies nothing).
4. **Hold publication until the verdict.** The one process failure in the source incident: the pipeline published the thin report first and judged second, so a misleading report was live for an hour. For an experiment run, the publish leg waits.
5. **Reject is a success.** The trial cost one research run and returned a receipt: the specific failure quoted, the rubric it failed, the date. That receipt prevents the same "surely the cheap model is fine for this" conversation from recurring evidence-free.

## Why the mechanical gate cannot save you

Any gate cheap enough to run on every output checks shape: sections present, keywords matched, minimum length. A weaker model produces exactly the artifact that satisfies shape while missing substance — not adversarially, but because shape is what it can always do and substance is what it sometimes skips. The gate's own documentation should say what it checks (`half-gate-whole-verdict`); the judge exists for everything the gate honestly disclaims.

## When to Use

- Any stage swap to a cheaper/local/faster model where the output feeds a real decision.
- Periodically re-run the trial: models change under their names, and a verdict is dated evidence, not a permanent law.

## When NOT to Use

- Mechanical stages with verifiable outputs (rename, reformat, extract-to-schema) — a deterministic check beats a judge there.
- When no unchanged comparison output exists — run one first; a judge with nothing to compare against grades on vibes too.

## Adjacent Patterns

- `green-tests-can-mirror-the-same-guess` — the judge-independence rule this borrows.
- `half-gate-whole-verdict` — claim exactly what the mechanical gate checks; the judge covers the rest.
- `verification-needs-a-negative-control` — the hazard-controls discipline whose absence the judge caught.

## Source

The personal agent `scripts/property_runner.py --gather-model` + the 2026-08-20 one-property trial: 70% vs 100% blind verdict, wrong-conclusion disqualifier (STR misread), diligence-not-method classification, publish-before-verdict named as the process bug, house re-run on the frontier model same hour.
