---
type: pattern
date: "2026-08-20"
source: The personal agent property-research lane — rounds 1-3 of the ten-house redo, 2026-08-19/21
tags:
  - agents
  - concurrency
  - reliability
  - cost
---

# Parallel Workers Share Nothing but the Tail

Long agent jobs parallelize safely when each job's entire data path is keyed by its own identity — and almost every pipeline still has one short shared tail (a manifest, a site build, a commit-and-push) where two finishers collide. Serialize only the tail, with a lock *inside the tool*; keep parallelism a **launch choice, never a pipeline feature**; and cap concurrency on burn-rate visibility, not on stability.

## The Problem

The personal agent's property lane researches houses in 30–40 minute worker runs, then publishes each to a shared static-site repo. Running two houses at once nearly halved wall-clock in round 1. But the publish step rebuilds the whole site (`rmtree` + regenerate), appends to one manifest, and commits/pushes one repo — two publishes racing would clobber each other's build and cross their pushes. And the shared tail hazard existed *before* any parallelism: a scheduler tick and a hand-run job could already meet there.

Two further findings from running it real:

- **The cost of parallel is burn rate, not stability.** Three concurrent research runs worked mechanically — and spent the account's remaining weekly quota three times faster, turning one usage-limit wall into three simultaneous mid-flight losses (~$31 of research thrown away in one moment). One or two at a time would have hit the same wall once, with two houses safely landed.
- **Orchestration is the part that rots.** A "parallel mode" built into the pipeline is permanent complexity serving occasional bulk work. Launching N detached single-job processes by hand (or by script) gets the same overlap with zero new moving parts, and sequential remains the default the scheduler runs.

## The Pattern

1. **Audit the data path per job.** Everything keyed by the job's own identity (its input file, its output file, its assets directory) is parallel-safe by construction. List what is NOT keyed: the manifest, the site build, the repo index, the push. That list is the tail.
2. **One lock, inside the tool, around the whole tail.** An `flock` on a lockfile in the shared repo, taken by the publish function itself — not by the caller, not by convention. Four lines. It also closes the pre-existing scheduler-vs-manual race, so it earns its place even if parallelism is never used again.
3. **Parallelism stays a launch choice.** The pipeline knows nothing about concurrency; a batch is N detached processes started together. Rolling back is "don't do that," not a code change.
4. **Cap concurrency on burn-rate, not stability.** The question is not "can the machine take it" but "how much spend is in flight when something external kills the round" — a usage limit, an outage, a revoked credential hits *everything currently running*. Two concurrent means one round-trip of loss; five means five.
5. **Same-key jobs never overlap.** The lock serializes the tail across different jobs; two runs of the *same* job would still fight over their keyed files. Idempotency/dedup upstream owns that.

## When to Use

- Bulk re-processing over a work-list of independent items with a shared publish/commit tail.
- Any pipeline where a scheduled tick and a manual run can already meet in the tail — take the lock even at concurrency one.

## When NOT to Use

- Items that genuinely share mutable state beyond the tail — that needs real design, not a lock.
- When the spend-per-job is trivial: below meaningful burn rate, just parallelize wide.

## Adjacent Patterns

- `detached-launch-from-timeboxed-agent` — how each parallel worker survives its launcher (and its harness).
- `at-least-once-needs-a-poison-ledger` — the idempotency layer that keeps same-key runs from meeting at all.
- `unattended-run-discipline` — the sequential default the scheduler keeps running.

## Source

The personal agent property lane, 2026-08-19/21: publish `flock` added before the first parallel pair; two-at-a-time pairs ran rounds 1–3 clean; the one three-wide launch hit the weekly usage limit and lost all three runs mid-flight, which set the burn-rate cap at two.
