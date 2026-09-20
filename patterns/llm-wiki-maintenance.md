---
type: pattern
date: "2026-05-09"
source: Andrej Karpathy's llm-wiki gist
tags:
  - wiki
  - memory
  - architecture
  - llm
---

# LLM Wiki Maintenance

Most LLM document workflows rely on RAG, retrieving context from raw files on the fly. The LLM re-derives answers from scratch every time. The LLM Wiki pattern replaces on-the-fly retrieval with persistent compilation. The LLM reads immutable raw sources and maintains a structured, interlinked collection of markdown files. The knowledge compounds.

## The Pattern

The architecture requires 3 layers:

1. **Raw sources**: Immutable files like articles, papers, or logs. The LLM reads these but never modifies them
2. **The wiki**: A directory of LLM-generated markdown files containing summaries, concept pages, and syntheses. The LLM owns this layer
3. **The schema**: A rules file (like `CLAUDE.md` or `AGENTS.md`) defining the wiki structure, workflows, and conventions

The LLM operates in 3 primary modes:

- **Ingest**: You drop a source into the raw directory. The LLM reads it, writes a summary, updates related entity pages, flags contradictions, and appends a record to `log.md`
- **Query**: You ask questions. The LLM reads the index, drills into relevant pages, and synthesizes an answer. If the answer is valuable, the LLM saves it back to the wiki as a new page
- **Lint**: The LLM scans the wiki for stale claims, orphan pages, missing cross-references, or contradictions, keeping the graph healthy

## Why It Works

- **Compounds knowledge**: Cross-references and syntheses persist. The LLM doesn't have to piece together fragments on every query
- **Near-zero maintenance cost**: Humans abandon wikis because bookkeeping is tedious. LLMs don't get bored updating cross-references or maintaining an index
- **Can handle conflicts explicitly**: When new data contradicts old claims, the LLM *can* flag the conflict in the wiki rather than silently retrieving conflicting chunks at query time — but only if `Lint` is treated as a first-class mode and the prompt asks for it. Left to default ingest, write-time synthesis tends to *smooth* contradictions into coherent prose instead (see Watch-outs)

## When to Use

- Personal knowledge bases or journaling
- Long-term research spanning weeks or months
- Internal team wikis fed by Slack threads, meetings, and project docs
- Any scenario where you accumulate knowledge and need it organized rather than scattered

## Watch-outs

- **Contradiction smoothing.** The wiki's readability is also its hazard: synthesis-on-ingest optimizes for a coherent narrative, and a genuine conflict ("eng says 12 weeks, sales promised 8") can get reconciled into one tidy sentence ("~10 weeks") with the strategic signal lost. Run a deliberate flag-don't-fix pass — see `contradiction-surfacing.md`. Don't rely on ingest to preserve tension on its own
- **Source-of-truth drift.** Karpathy keeps raw sources immutable for a reason, but in practice people stop opening them and start trusting the AI's summary. Errors then compound: a wrong line seeds the next answer. Enforce "raw is authoritative, wiki is derived" as a rule, not a hope. A *regenerable* compiled view (rebuilt from a structured layer, never hand-edited) sidesteps this — see `memory-substrate-selection.md`
- **Scale ceiling.** Karpathy's own stated sweet spot is ~100–10k high-signal documents. Above that you need extra search tooling, and precise structured queries ("every item matching X, sorted by date") are not what a folder of markdown does well
- **Single-writer assumption.** The pattern presupposes one agent writing one set of files. Multiple agents writing markdown concurrently collide. Past that point, a structured store is the substrate — see `memory-substrate-selection.md`

## Measured case: playbooks rewritten by the run that used them (the personal agent, 2026-08-20)

A compiled knowledge layer works hardest when its maintainer is *the run that just exercised it*.
The personal agent's property-research workers read a per-county "playbook" (GIS endpoints, portal recipes, code
platforms, dead ends) before researching and rewrite it after — under three rules: fix-or-delete a
failed row in place, never append a contradicting note (`appending-is-not-learning`); hard per-file
line cap; recipes only, no case data. One day of real runs produced the full loop: workers replaced
hand-seeded guesses with exercised recipes, marked two dead endpoints as dead ends with the working
alternative, documented *why a previous run had reached a wrong conclusion* (a federal service that
returns a silent zero unless every field is requested), and one worker corrected a wrong diagnosis
in a *sibling county's* file. Use-time verification substitutes for the refresh ritual: the file is
re-validated every time it is worth reading, precisely because the reader is about to bet a paid run
on it. One measured caveat: the first warm run spends its savings on *depth* (cracking portals the
cold run never breached) rather than dollars — the cost drop shows on the second warm run, once the
file holds exercised recipes rather than seeds.

## Adjacent Patterns

- `memory-substrate-selection.md` — when to use this write-time wiki versus a query-time structured store, and the hybrid that beats both
- `contradiction-surfacing.md` — the rigorous form of the `Lint` mode; flag tensions, never resolve them
- `three-tier-memory-pipeline.md` — orthogonal hot/living/session-log layering that applies on top of this substrate

## Layer 1's immutability needs a mechanism, not a sentence

"The LLM reads these but never modifies them" is the invariant the whole compilation model rests on. The wiki is only as trustworthy as the captures it compiles from, and a rewritten capture silently re-grounds every claim citing it — including the claims of any guard that validates a synthesis *against* that capture, which will keep passing while the ground moves.

It is also the hardest layer to protect, because raw capture must stay writable to work. The household agent's wiki put the filesystem immutable bit on the synthesis layer and the wiki root and deliberately left `raw/` writable; the append-only rule for `raw/` lived in a documentation file. A scheduled sync job authored a year later hardcoded one capture filename and instructed its agent to overwrite it. Two upstream revisions each destroyed the prior capture before anyone looked. Nothing was careless in isolation — the job's author read the schema, not the prose rule, and no check in the system could disagree.

Two mechanisms, in order of leverage:

1. **Derive every capture filename from its capture date; never hardcode one.** A fixed name is an overwrite waiting for a second revision. Date-stamping makes append-only structural — there is no name to overwrite — and it costs one function.
2. **Detect rewrites with version control, not a bespoke hash.** Keep the raw layer in git and ask `git log --diff-filter=M -- raw/` which captures changed after they were committed, plus `git status --porcelain` for one that has not been committed yet. A `sha256` recorded in a state file alongside each capture looks like the natural instrument and is not: unread, its meaning drifts, and on this wiki it produced 2 false positives out of 7 on its first real run. See [[instrumentation-with-no-reader]].

Wire the result into whatever already reaches a human. A capture rewritten in place is invisible by construction — that is what makes it worth a scheduled check rather than a rule.

## Source

- [Andrej Karpathy's llm-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
