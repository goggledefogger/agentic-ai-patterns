---
type: pattern
date: "2026-07-21"
source: The personal agent day-walk burst — 8 PRs cut off main in one evening (#34–#40, #43), reviewed and consolidated 2026-07-21
tags:
  - git
  - workflow
  - agents
---

# Parallel Branch Burst: N Branches off One File Is Conflict Debt, Not Throughput

An agent-assisted evening produced eight feature PRs, each cut independently from the same base commit, five of them editing the same two files. It *felt* like throughput — eight green branches by midnight. It was debt: two PRs defined the same function with different bodies, one duplicated an endpoint another already wired, and merging required hand-resolving three multi-hunk conflicts, re-running suites per merge, and closing one PR outright. Review capacity, not generation capacity, was the bottleneck — and every parallel branch borrowed against it.

The failure is structural, not a discipline lapse: branches cut from the same base **cannot see each other**. Each one re-implements whatever shared plumbing it needs (a store, a helper, a route table entry), and the collisions only surface at merge time, where the hardest version of the integration work has to happen — by whoever merges, not whoever generated.

## The Pattern

Parallelize by *file ownership*, serialize by *file contention*:

- **Before fanning out**, split the work so no two branches touch the same file. If a clean split is impossible, the work isn't parallel — it's a stack (each branch based on the previous) or a queue (merge each before cutting the next).
- **Land the trunk feature first.** Satellites cut *after* the core merges see the real API and extend it instead of reinventing it. The five colliding PRs would have been trivial diffs on top of the merged core.
- **Cap WIP by review capacity:** open no more PRs in a burst than will be reviewed before the next burst. Ten unreviewed PRs age like milk — the base moves, `mergeable` rots, and honest code turns into conflict work.
- Merging an aged burst anyway: merge trunk-outward (core first, satellites resolved against fresh main one at a time, suites re-run per merge), and expect to close duplicates rather than union them.

Agent note: generation is now so cheap that "one branch per idea" is the default failure mode of an enthusiastic session. The prompt-level fix is to hand the agent a *work list with file boundaries*, not a theme.
