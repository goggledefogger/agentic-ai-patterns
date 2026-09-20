---
type: pattern
date: "2026-08-20"
source: A course architecture doc, 2026-08-20 — "tempting to make it a skill that dynamically outputs the document so it's always up to date, but not sure." The split shipped instead, and the checker caught a false claim on its first run.
tags:
  - documentation
  - drift
  - generated-files
  - decision
---

# Split the Doc by Verifiability, Don't Generate It

When a doc must "never get out of date," the tempting fix is a skill or script that regenerates the whole thing on demand. Resist it. Split the doc by what can be *verified* instead: judgment stays hand-written prose, checkable facts go in a machine-parseable section a script verifies against reality. The doc stays a reviewable artifact; the script keeps it honest.

## The Problem

An architecture doc, a stack overview, a systems map — these rot, and everyone knows it, so someone proposes generation: "have the AI output it fresh each time, then it's always current." That fails twice:

1. **The valuable half isn't derivable.** What each piece is *for*, what's deliberately separate, what talks to what and why — that's judgment. A generator re-deriving it from the filesystem produces confident slop that reads fine and is subtly wrong, which is worse than stale prose because nobody distrusts it yet.
2. **A doc that doesn't exist until rendered can't be reviewed.** A collaborator can't PR an edit to output. There's no history of *decisions*, only of the generator. The doc stops being a shared artifact and becomes one person's script.

Meanwhile the half that actually rots — paths, repo names, deploy targets, URLs, ports — is exactly the half a script *can* check, and nobody writes the checker because the generation debate ate the energy.

## The Pattern

1. **Prose carries judgment, hand-written.** The map, the why, the deliberate separations. This half is allowed to change slowly, because it does.
2. **Facts live in one machine-parseable section** — a markdown table with typed columns (local path · git remote · URL). `—` skips a cell.
3. **A small checker verifies the table against reality**: paths exist, remotes match `git remote -v`, URLs answer on a closed list of accepted codes (200/301/302/401 in the reference implementation, an auth wall is a live URL). `--check` exits nonzero for CI or hooks.
4. **A same-commit rule covers what the checker can't:** the script catches *rot* (a deleted path, a dead URL) but only a human can add the *new* piece. So the contract is: structural change → update the doc in the same commit. Write the rule where commits happen (CLAUDE.md), not in the doc alone.

"Never out of date" honestly means "drift is detected within a day and additions are a named obligation." No doc self-updates; say so plainly in the doc's header.

## Why It Works

The split matches each half's failure mode to its remedy. Prose fails by *judgment drift* — only review catches that, so it must stay a reviewable artifact. Facts fail by *reality drift* — only a check against reality catches that, and a script does it better than any reader. Full generation applies the facts-remedy to the judgment-half (slop) and no remedy to the facts-half between regenerations.

Proof from the source case: the checker's **first run** caught a row claiming a directory was a clone of a public repo — it was actually a folder inside the parent repo, synced outward by a script. The claim had read fine for months. The prose around it needed no change; one table cell did.

## When NOT to use it

- The doc is *all* facts (a port registry, an inventory) — then full generation is right; see `churn-free-generated-artifacts.md` for its diff hygiene.
- The doc is *all* judgment (an ADR) — nothing to check; see `decision-doc-adr.md`.
- Nobody will maintain a checker — then `doc-warning-preamble.md` is the honest floor.

## Related

- `self-reporting-staleness-check.md` — the recurring-job altitude of the same instinct, over a whole corpus rather than one doc's fact table
- `living-doc-refresh-ritual.md` — layer-splitting by *stability*; this pattern splits by *verifiability*, and the two compose
- `a-dry-run-that-derives-will-drift.md` — the same echo-vs-derive fork, one level down
