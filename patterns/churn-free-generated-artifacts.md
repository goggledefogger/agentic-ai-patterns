---
type: pattern
date: "2026-06-13"
source: A course participant's home-agent system (the workspace repo BUILD-STATUS churn fix, PR #106; build_status.py)
tags:
  - automation
  - git
  - signal-noise
  - generated-files
---

# Churn-Free Generated Artifacts

When a script generates a file that's committed to git, design the *representation* so that only a material change produces a diff. Mechanically discard noise-only changes. The git history of the artifact then becomes a real changelog of meaningful events instead of a wall of bot commits.

## The Problem

Auto-generated status files — build dashboards, daily digests, sync registries, published indexes — tend to change on *every* regeneration even when nothing important happened: a "generated at" timestamp, a process ID, a "days since X" counter that ticks daily, the bot's own previous commits showing up in a recent-commits list. So the file commits constantly, and its git history fills with `chore: regenerate (automated)` noise.

Two costs, both real:

1. **Signal drowns.** When you actually need to know "what changed in the build status this week," you can't see it — the meaningful diffs are buried among hundreds of timestamp-only commits. The thing that was supposed to give visibility destroys it.
2. **The file lingers dirty.** A working tree that's always showing a modified generated file trains you to `git add -A` blindly, which is how unrelated changes get swept into commits.

Concrete instance: That system's `BUILD-STATUS.md` regenerated daily via launchd. Bot commits "dominated this repo's history" because the snapshot always changed — days-since-push counters, PIDs, last-run timestamps, the `_Generated_` line, and the bot's own commits appearing in the recent-commits section. The signal (a task actually stopped running) was invisible inside the noise (everything ticked one day forward).

## The Pattern

Don't stop generating, and don't stop committing. Redesign what the artifact *renders* so that a healthy, unchanged system renders **byte-identical** to yesterday — and only a meaningful state change produces a diff.

### Render stable state as stable bytes

Work through each field that changes on regeneration and ask "does this carry signal, or is it just the clock ticking?" Then neutralize the clock:

| Noisy field | Churn-free rendering |
|---|---|
| `Generated at 2026-06-13 02:45` | Drop it, or only bump it when other content changes |
| `Days since last push: 4` (ticks daily) | Drop the counter; keep the *date* of last push (stable until a new push) |
| Process ID `48213` | Render as a presence checkmark `✓` (the PID's value carries no signal; its existence does) |
| `Last run: 2026-06-13 02:45:01` | Bucket to a band: `current (≤48h)`. Only render a literal date once it goes stale |
| Bot's own commits in a recent-commits list | Exclude commits by the generating identity |

### Preserve the signal you actually want

The point isn't "stop changing" — it's "change *only* on signal." Keep stall/regression detection intact: a task that stops running flips from `current (≤48h)` to a dated cell, which **is** a material change, so it commits. You lose the noise, not the alarm.

### Make the regen step self-cleaning

The generator should leave the working tree clean when nothing material changed: diff its fresh output against the committed version, and if the only differences are timestamp-band / PID-presence noise, **revert the file** rather than leave it dirty. Then a `git status` that shows the artifact modified actually means something happened.

## Sub-pattern: guard intentionally-divergent files against "helpful" consolidation

A close cousin of churn is the well-meaning cleanup that *merges things that are different on purpose*. In the same PR, two `config.json` files that were divergent by design each got a `_purpose` key explaining why they differ "so nobody 'fixes' them into one."

When two artifacts look like duplicates but aren't, leave an in-file marker stating the intent. You're defending against a future agent or teammate whose instinct is to DRY them up — the same instinct that's usually right, which is exactly why it needs a tripwire here. This defends against false-positive cleanup the way a churn-free render defends against false-positive diffs.

## Anti-pattern: solving churn by not committing

Tempting shortcut: stop committing the generated file at all (gitignore it, regenerate on read). Sometimes correct — but you lose the audit trail. If the artifact's *history* is valuable (when did this task stop running? when did that doc go stale?), you want it committed; you just want it committed *only when it matters*. Churn-free rendering keeps the history and kills the noise. Reach for gitignore only when the history genuinely has no value.

## How to Adopt

1. List every field your generator emits. For each, decide: signal or clock-tick?
2. Re-render clock-ticks as stable bytes (drop, bucket to a band, or collapse to a presence marker).
3. Exclude the generating identity's own commits from any commit/history list the artifact renders.
4. Add a self-clean step: if the fresh output differs from committed only in noise, revert instead of committing.
5. Verify the alarm still fires: simulate a real state change (stop a task, stale a date) and confirm it *does* produce a diff and commit.

## Adjacent Patterns

- **Registry-based monitoring** (`registry-based-monitoring.md`) — the generator that produces these status artifacts; churn-free is how you keep its output readable over time.
- **Vault-as-CMS publisher** (`vault-as-cms-publisher.md`) — published galleries/indexes are prime churn sources (re-sorted lists, image hashes, build timestamps). Apply this so the published diff means something.
- **Atomic state writes** (`atomic-state-writes.md`) — about *correct* writes; this is about *quiet* writes. Pair them for generated state files.
- **Self-reporting staleness check** (`self-reporting-staleness-check.md`) — both turn a git-tracked file's diff into signal: staleness adds signal, churn-free removes noise.
