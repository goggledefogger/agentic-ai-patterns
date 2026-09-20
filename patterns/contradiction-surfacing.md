---
type: pattern
date: "2026-05-29"
source: Nate Jones / OpenBrain critique of the write-time wiki pattern; Karpathy llm-wiki "lint" mode
tags:
  - memory
  - wiki
  - lint
  - contradictions
  - decision-support
---

# Surface Contradictions, Don't Resolve Them

A write-time wiki's worst failure mode is that it resolves a real contradiction into smooth, confident prose. The classic shape: engineering's notes say the build is 12 weeks, sales promised the client 8, and a well-meaning synthesis writes "~10 weeks." The number reads cleanly and the **strategic signal — that two parts of the org disagree — is gone.** The gap between what one side knows and what the other promised was exactly the thing leadership needed to see, and the wiki smoothed it away.

This is the same trap as a dashboard versus a spreadsheet: the condensed view is easier to read precisely because it drops detail, and it can drop the one thing you needed. The antidote is a deliberate pass whose entire job is to **flag tensions and never fix them.**

## The Problem

Synthesis-on-ingest optimizes for a coherent narrative. Coherence and contradiction-preservation are in tension: the more readable the page, the more likely a genuine conflict got reconciled into one tidy sentence. And because the prose is confident, the reader doesn't question the gap they can't see. A structured store has the opposite problem — it stores both conflicting facts faithfully but is not contradiction-*aware*, so the tension sits silently in adjacent rows until someone asks exactly the right question.

Either way, contradictions don't surface themselves. You need a pass built for it.

## The Pattern

A read-only audit/lint pass over the load-bearing state of the system. It is **a distinct mode, not part of ingest** — in Karpathy's gist it's the wiki's "lint" mode; in OpenBrain it's shipped as a "contradiction audit" plugin over the database. Substrate-agnostic; the discipline is the same.

**Classify, don't conclude.** Surface findings in categories:

- **Hard conflict** — two assertions that cannot both be true (two stated deadlines for the same obligation; an assumption in one plan that violates a constraint in another).
- **Assumption drift** — a plan assumes X, but X changed in its source-of-truth file and the dependent plan was never updated.
- **Stale-dependency** — a decision rests on data whose source is now materially stale (pairs with freshness-timestamp discipline: a plan built on old numbers is itself suspect).
- **Timeline collision** — deadline/sequencing problems across domains (a coverage cliff lands before the thing meant to cover it is expected to exist).

**Flag-don't-fix is the whole point:**

- Never edit a source file, never reconcile, never pick a winner, never invent a compromise figure. The tension goes back to the humans intact.
- **Zero findings is a valid, honest result.** State "no contradictions surfaced" plus any freshness flags. Do not manufacture tension to look productive.
- **Name what was skipped.** If sources were unreadable, gated, or out of scope, list them — an audit that implies completeness when it was partial is worse than no audit.

## Restricted-Content Discipline

When the state files live in sensitive tiers, the audit reads across them and therefore:

- Runs only under explicit authorization — not implicitly on every session.
- Never routes restricted content to cloud/MCP; it's local reasoning over local files.
- Produces output that inherits the **highest** sensitivity of any source it touched. Terminal-first; persist only at that tier.

See `sensitivity-tiered-access-control.md` for the gate this rides on.

## When to Use

- Before a high-stakes decision or meeting where cross-domain assumptions are about to drive a real choice.
- After fresh data lands, to check whether new numbers broke an assumption a plan elsewhere depends on.
- As a periodic deliberate sweep of a knowledge base where plans in different areas quietly reference each other.

## When NOT to Use

- Inside the ingest/compile step — keep it a separate pass so synthesis pressure can't suppress the flags.
- On a corpus with no cross-references between areas; there's nothing to collide.
- As an every-session reflex when it reads sensitive content — invoke it on purpose.

## Watch-outs

- **False positives from legitimate divergence.** Two areas can hold different assumptions *on purpose* (different time horizons, deliberately hedged scenarios). Default to flag-don't-resolve, but expect to tune the threshold for "contradiction worth surfacing" against real runs — the first real use is the calibration.
- **Staleness feeds it but isn't itself a contradiction.** A stale source is a *reason to distrust* a dependency, not proof of conflict. Keep the categories distinct so "old data" doesn't get reported as "hard conflict."
- **It degrades into noise without flag-don't-fix discipline.** The moment the pass starts proposing reconciled answers, it becomes another smoothing synthesis — the exact thing it exists to prevent.

## Adjacent Patterns

- `llm-wiki-maintenance.md` — this is the rigorous form of that pattern's "lint" mode; the wiki oversells "handles conflicts" without this counter-discipline
- `memory-substrate-selection.md` — preserving contradictions is a core reason a structured/faithful layer beats a pure synthesis layer
- `system-understanding-protocol.md` — shares the staleness-flag and atomic-citation discipline; "the wiki said so" isn't a source
- `living-doc-refresh-ritual.md` — refreshed sources reduce the stale-dependency category at the root
- `sensitivity-tiered-access-control.md` — the gate the audit runs under when sources are restricted

## Source

- [Nate Jones / OpenBrain (OB1)](https://github.com/NateBJones-Projects/OB1) — the contradiction-smoothing critique of write-time wikis and the contradiction-audit-as-plugin framing
- [Andrej Karpathy's llm-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — "lint" as a first-class wiki mode
