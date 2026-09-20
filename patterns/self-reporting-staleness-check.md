---
type: pattern
date: "2026-06-13"
source: A course participant's home-agent system (the workspace repo truth-sync PR #107 — the weekly-maintenance task "Source 6")
tags:
  - documentation
  - drift
  - governance
  - automation
---

# Self-Reporting Staleness Check

Build a recurring job that detects drift in your *own* corpus — stale paths, references to renamed or archived sources, claims contradicted by current reality, dates past their revisit-by — and reports it. The corpus tells you it's stale before a reader trusts a stale line.

## The Problem

Every other drift pattern in this library is *passive*. `living-doc-refresh-ritual.md` says refresh the doc by verifying against the running system — but a human has to decide to refresh. `doc-warning-preamble.md` says annotate the doc with a "verify before trusting" warning — but the reader still has to heed it. Both depend on someone noticing the drift first.

The failure mode this leaves open: drift you don't know about yet. A skill points at a file that got renamed. A note cites a vault path from before a migration. A README references a repo that's been archived. A "revisit by 2026-05-07" deadline that quietly passed. Nothing is *wrong* on the screen — the line reads fine — so nobody refreshes it, and an agent reading it as truth acts on a lie.

Concrete instance: a `/catch-up` workflow whose step pointed at `an archived repo's ROADMAP.md`. That repo was archived months earlier; the live roadmap had moved to `ROADMAP.md`. The catch-up ran clean and produced a confident summary — framed entirely off the stale copy. The drift was invisible until someone cross-checked by hand. The fix isn't "remember to check the path." The fix is a job that greps for "references to archived repos" and prints them every run.

The altitude ladder, completed:

> notice it → document it (CLAUDE.md rule) → enforce it (pre-push hook) → **auto-detect it (this pattern)**

Will climbed each rung as he hit the same failure class. The truth-sync PR didn't just fix the stale specs by hand — it added a weekly-maintenance check (`Source 6`) for brief-refresh age, doc-vs-status drift, and "unsuperseded Plow / old-vault references, so this staleness becomes self-reporting."

## The Pattern

A check that runs on an existing cadence (session start, weekly cron, pre-commit) and scans the corpus for drift signals. Three tiers of signal, cheapest first:

### 1. Zero-config signals (work in any vault, no setup)

- **Broken wikilinks** — `[[target]]` where no note named `target` exists. The single highest-value universal check; catches renames and deletions immediately.
- **Past-due revisit dates** — frontmatter or inline `revisit_by:` / `review by:` / `revisit by:` dates earlier than today. Surfaces the deadlines you set for yourself and forgot.

### 2. Config-driven signals (one line of setup per known hazard)

A small rules file (`.staleness-rules.txt`) of `regex => message` pairs, one per known drift hazard for *this* corpus:

```
~/Documents/Obsidian/ParticipantVault  => old vault path; canonical is ~/Documents/ParticipantVault since 2026-06-01
a private org/ROADMAP              => archived repo; live roadmap is ROADMAP.md
plow-workspace               => superseded by Hermes 2026-04-20
```

These are the corpus-specific lies. You add a rule the moment you make a migration, so the *next* stale reference reports itself.

### 3. Cross-source signals (when you have a second source of truth)

- References to repos that are archived/renamed on the remote (compare against `gh repo list`).
- Doc status claims vs. an authoritative generated artifact (BUILD-STATUS, a manifest).

### Output is a report, not a failure

Staleness is advisory by default — print a markdown drift report at the top of the session/run; don't hard-fail. (Contrast with a *preflight* gate, which hard-fails before a destructive migration — see `smoke-tests-with-real-data.md`.) The report's job is to put the drift in front of whoever is about to trust the corpus, before they trust it.

## Anti-pattern: the check that needs its own maintenance

If the rules file grows unbounded, it becomes another stale corpus. Keep it to *active* hazards: when a superseded reference is fully purged from the vault, delete its rule. The check should shrink as you clean up, not accrete forever.

## Anti-pattern: running every check in every vault

The broken-wikilinks check is the right zero-config default for a *clean knowledge vault* — but it's noise in a *high-velocity work vault* where forward-links to not-yet-created notes are routine, and where transient notes (briefs, drafts) get deleted after sending so links to them dangle by design. Observed: the same checker reports 0 broken links in a curated advice vault and 179 in a recruiting/ops vault — almost all of them intentional forward-links and deleted briefs, not rot.

The fix isn't to make broken-wikilinks smarter; it's to run only the checks that earn their keep *in that vault*. The reference script takes a `--checks` subset for exactly this: a forward-linking-heavy vault runs `--checks past-due-dates,stale-references` and skips the wikilink noise, while a curated vault runs all three. Match the check set to the vault's conventions, not the other way around.

## Anti-pattern: hard-failing on advisory drift

A broken wikilink shouldn't block a commit — sometimes you create the link before the note, deliberately (this library does it). Run staleness as a report. Reserve hard-fail for genuine preflight gates (working-tree-clean, schema-valid) where proceeding corrupts data.

## How to Adopt

1. Drop a portable `vault-staleness-check` script into `scripts/` (reference implementation ships with this guide; it's stdlib-only and zero-config for tiers 1–2).
2. **Wire it** (see `wire-into-existing-flows.md`) — call it from the top of your `/catch-up` or session-start flow so the drift report prints on a normal day. A staleness checker nobody runs is itself a standalone artifact.
3. Add a `.staleness-rules.txt` rule **at the moment you make a migration** — rename a repo, move a vault, supersede a runtime. That's when you know the hazard; that's when the rule is cheap to write.
4. When you fully purge a superseded reference, delete its rule.

## Adjacent Patterns

- **Living-doc refresh ritual** (`living-doc-refresh-ritual.md`) — the manual rung below this one. The staleness check tells you *when* the living layer needs the ritual.
- **Doc-with-warning-preamble** (`doc-warning-preamble.md`) — the passive rung. Use both: the preamble warns the reader, the check finds the drift the preamble can't predict.
- **Wire adoptions into existing flows** (`wire-into-existing-flows.md`) — a staleness check only earns its keep if a forcing function fires it.
- **Registry-based monitoring** (`registry-based-monitoring.md`) — same instinct (auto-discover and report) applied to skills/tasks instead of doc drift.
