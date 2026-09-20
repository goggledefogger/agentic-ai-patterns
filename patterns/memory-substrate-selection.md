---
type: pattern
date: "2026-05-29"
source: Andrej Karpathy's llm-wiki gist + Nate Jones / OpenBrain (OB1) comparison
tags:
  - memory
  - architecture
  - wiki
  - structured-data
  - decision
---

# Choosing a Memory Substrate — Wiki vs Structured Store

Two memory patterns went viral in 2026 and they force one decision: **when does the AI do the hard thinking — when information comes in, or when you ask about it?** Everything else follows from that fork. Karpathy's `llm-wiki` does the work at **write-time** (ingest compiles understanding into cross-referenced markdown). Nate Jones's OpenBrain does it at **query-time** (faithful structured storage, synthesis on demand). Picking the wrong one for your situation is one of the more expensive context-layer mistakes you can make, and the failure is quiet.

This pattern is the decision framework. The companion `llm-wiki-maintenance.md` documents the write-time wiki itself; this file is about when to reach for it versus a structured store, and the hybrid that often beats both.

## The Fork

**Write-time / file-native wiki** (Karpathy). The agent is a *writer*. On ingest it reads a source, summarizes it, updates topic pages, builds cross-references, flags contradictions. The hard work happens once, up front. Afterward, browsing is cheap — answers are pre-compiled. Substrate: a folder of markdown, Obsidian as the read surface, raw sources kept immutable alongside. You own files.

**Query-time / structured store** (OpenBrain / OB1). The agent is a *reader*. Ingest is lazy and cheap — tag a row, done. The hard work happens at query time, reconstructed fresh from structured data each time. Substrate: a database (OB1 is PostgreSQL + pgvector, default-deployed on Supabase, exposed to multiple AI clients through an MCP gateway with row-level security). You own a database.

Neither is "better." They are good at different things, and the difference is *where the compute lands*, not quality.

## The Decision Axes

Score your situation against these. A clear lean on the first three usually settles it.

| Axis | Lean **file-native wiki** when… | Lean **structured store** when… |
|---|---|---|
| **Scale** | ~100–10k high-signal docs (Karpathy's own stated range) | thousands–millions of rows; filtered/sorted queries across them |
| **Writers** | one agent, one (or asymmetric) human | multiple agents/tools writing concurrently (markdown files collide; a DB handles simultaneous writes) |
| **Readers** | the people who read are technical *or* a note app is enough | non-technical readers and tools need a stable query API |
| **Content shape** | narrative, evolving understanding, value is in the *connections between sources* | discrete facts, value is in *precise structured operations* ("every deal > $50k last quarter") |
| **Sensitivity / locality** | local-first, restricted data must not leave the machine, no SaaS dependency | central shared store is acceptable; a managed-DB SaaS in the loop is fine |
| **Provenance need** | "the synthesis represents my thinking" is good enough | every claim must trace to a timestamped source on demand |

Two traps to name explicitly:

- **Do not adopt DB infrastructure below the scale where files break.** Standing up Postgres/Supabase/edge-functions to hold 150 notes is solving a problem you don't have and adding a thing to babysit. Files win until they visibly stop winning.
- **A managed-DB "memory" is not local-first.** If your differentiator is "restricted content never crosses the network," an MCP-gateway-over-cloud-DB is in direct tension with that, and with any rule that forbids granting MCP servers scope over sensitive folders. The file-native substrate is the one that keeps that promise.

## The Hybrid — Don't Pick, Compile

The strongest setup is usually neither alone: **a structured source of truth with a compiled, regenerable view on top.** Nate's "graph over OpenBrain" is the SQL version; the file-native version is the one most Obsidian + Claude vaults should reach for:

- **Structured layer = source of truth.** Frontmatter as queryable metadata, plus a *scripts-produce-derived-files* discipline for any tabular data: raw export stays immutable, a deterministic script computes derived artifacts, and the numbers in any summary are traceable to the script — never narrated from memory. New information always lands here first.
- **Compiled view = a generated artifact, never hand-edited.** Topic summaries / dashboards / a daily picture, regenerated from the structured layer on a schedule or on demand. Because it is always rebuilt from ground truth, it **cannot drift** — and crucially, **errors don't compound**: a wrong line in a hand-edited wiki seeds the next answer; a regenerated view just gets fixed at the source and rebuilt.
- **Read either, depending on the question.** Compiled view for the synthesized narrative; structured layer for the precise fact with provenance.

This gets you the wiki's browsability and the database's faithfulness without the wiki's drift or the database's headlessness. Obsidian stays the read surface either way — even OB1 users bolt Obsidian on top, because a database has no good human browse layer of its own.

## When to Use

- You're deciding the memory substrate for an agent and want the choice to be deliberate, not accidental.
- Someone proposes migrating a working markdown vault onto a database (or vice versa) and you need to evaluate whether the substrate actually buys anything.
- You want the hybrid and need to know which layer owns what.

## When NOT to Use

- The decision is already forced by a hard constraint (a non-technical user who only has a note app; restricted data that can't leave the machine) — then there's nothing to weigh, file-native wins by constraint, skip the framework.
- One-shot or short-lived agents with no second session to remember into.

## Watch-outs

- **Staleness is asymmetric, and the dangerous one is invisible.** A neglected database looks like *ignorance* — gaps are visible, old facts are still true. A neglected write-time wiki looks like *confident misinformation* — pages still read authoritatively while the synthesis silently goes wrong. Budget refresh accordingly; the wiki needs it more.
- **The source of truth quietly migrates.** Karpathy keeps raw sources immutable for exactly this reason, but in practice people stop going back to them and start trusting the AI's summary. Whichever substrate you pick, make "the raw/structured layer is authoritative, the synthesis is derived" an enforced rule, not a hope.
- **The schema/instructions file is the highest-leverage document either way.** It tells the agent how to organize and synthesize. Under-invest in it and the whole memory layer is worse than it should be — not because the substrate can't be good, but because nobody told it how.

## Adjacent Patterns

- `llm-wiki-maintenance.md` — the write-time wiki substrate in detail (the file-native half of this fork)
- `three-tier-memory-pipeline.md` — orthogonal layering (hot / living / session-log) that applies on top of *either* substrate
- `contradiction-surfacing.md` — the audit pass that protects the cross-domain reasoning a compiled view would otherwise smooth
- `sensitivity-tiered-access-control.md` — why restricted content forces file-native + no-MCP-scope, a constraint that can settle the fork on its own
- `local-model-routing-for-restricted.md` — the local-first companion when restricted content must never cross the network
- `living-doc-refresh-ritual.md` — anti-drift for the compiled-view layer

## Source

- [Andrej Karpathy's llm-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Nate Jones / OpenBrain (OB1)](https://github.com/NateBJones-Projects/OB1) — PostgreSQL + pgvector + MCP-gateway structured store; the "graph over OpenBrain" hybrid proposal
