---
type: pattern
date: "2026-07-25"
source: The personal agent — a life-planning vault `Initiatives/` promotion, stale registry pointer found 2026-07-24
tags:
  - drift
  - registry
  - migrations
  - verification
  - anti-pattern
---

# A Stale Pointer Doesn't Go Quiet, It Asserts — and Cleanup Can't Cross Repos

When a resource reorganizes itself, the copies of its layout that live in *other* repos do not break. They keep answering, confidently and wrongly. Worse, a pointer written carefully enough to warn against the wrong answer will keep issuing that warning after the wrong answer has become the right one.

The migration's own cleanup pass cannot help. Its blast radius is one repo; the stale copy is outside it, by construction.

## The Problem

`migration-blinds-readers.md` covers the case where a store changes shape and a *reader* goes silent — the symptom is emptiness, and the trap is trusting it. This is the other failure:

|  | `migration-blinds-readers` | This pattern |
|---|---|---|
| What broke | a tool that reads the store | a *fact about the store* copied elsewhere |
| Symptom | silence, empty results | a confident, specific, wrong answer |
| Detected by | noticing suspicious quiet | nothing — it looks like knowledge |

Silence at least *feels* like a missing answer. A stale pointer feels like expertise, so it gets acted on, cited, and — the compounding part — **copied forward into new work**.

Concrete instance (the personal agent, 2026-07-24). The personal agent's registry recorded a vault's initiative folder as `Life/initiatives/<slug>.md`, and — because an earlier reconnaissance had been confused by the obvious name — added an explicit caution: *"not a top-level `Initiatives/`, which is why a recon looking for the obvious name missed it."* The vault had since promoted that folder to exactly `Initiatives/` at root. So the registry pointed at a path that no longer existed **and** warned against the one that did, in two files, for over a week. That morning the personal agent had appended new aliases to that same row, propagating the dead path further.

The vault had already run its own cleanup: a commit literally titled *"close the gaps the `Initiatives/` promotion left open."* It could not reach the personal agent's copy. Different repo, no link, no awareness the duplicate existed.

It surfaced only because a dispatched worker was told **"if a convention here conflicts with the vault's own docs, the vault wins"** — so it read the vault's `CLAUDE.md`, filed correctly, and *reported the conflict* instead of silently obeying the brief.

## The Pattern

1. **Give every dispatched worker an explicit precedence rule**, naming the resource's own docs as authority over the brief, and require it to *report* conflicts rather than silently resolve them. This is the only mechanism in the whole loop that can detect this class of drift — the orchestrator cannot self-diagnose a wrong belief, and the resource cannot see the copy.

2. **Hold the question, not the answer.** The registry never needed the folder path. "Initiatives live in this vault — ask the vault where" would have been correct through the reorg and every future one. A pointer that resolves *at use time* cannot go stale; a cached path can only rot. Ask of every recorded detail: **would this still be true if the resource reorganized tomorrow?** If no, it is a finding, not a pointer, and it belongs downstream.

3. **Treat a confident negative as the highest-risk sentence you can write.** "It is X, and specifically **not** Y" is doubly wrong when it inverts, and it actively suppresses the correct search. Prefer "resolve via the vault's own docs" over encoding the outcome of one past investigation as a standing warning.

4. **When you do correct one, say it was wrong.** A silently-fixed pointer teaches nothing and the next session re-derives the same confidence. Leave the inversion visible in the note.

## Why It Works

- **Precedence rules turn workers into drift sensors.** A worker that touches the real resource is the only actor with both the stale belief and the live truth in hand at once. Told to prefer the resource and report conflicts, it detects for free what no scheduled audit was watching for.
- **Use-time resolution has no staleness surface.** You cannot hold a wrong path if you hold no path.
- **It bounds what cleanup must reach.** If cross-repo copies are questions rather than answers, a reorg's blast radius genuinely is one repo again.

## Watch-outs

- **The careful note is the dangerous one.** Casual prose gets re-verified; a note with a dated citation and a hard-won caveat reads as settled and gets trusted for longer.
- **Agreement between two copies is one witness, not two** — copies rot together (`self-reporting-staleness-check.md`).
- **Appending to a stale row inherits its staleness.** Adding a correct fact to a row whose pointer is wrong ships the wrong pointer with fresh credibility and a new date.

## Adjacent Patterns

- `migration-blinds-readers.md` — the silent-reader half of the same migration hazard.
- `self-reporting-staleness-check.md` — finds drift within a corpus; this is drift *across* corpora, where that check cannot see.
- `external-store-as-source-of-truth.md` — the discipline that prevents it: one canonical home, everyone else queries.
- `reach-surface-resolution.md` — resolve the surface, don't infer from a cached detail.
- `merge-then-update-consumers.md` — the general duty this instance shows you cannot always discharge, because you may not know who your consumers are.

## Source

The personal agent `registry/aliases.md` + `registry/resources.md`, both asserting `Life/initiatives/` and warning against `Initiatives/` after the vault had promoted the folder to root. Corrected 2026-07-24 in commit `861a905`, found by a dispatched worker obeying a "the vault wins" precedence instruction.
