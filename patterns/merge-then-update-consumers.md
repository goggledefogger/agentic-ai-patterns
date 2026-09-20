---
type: pattern
date: "2026-03-27"
source: A course participant's home-agent system (skills repo refactoring pattern)
tags:
  - refactoring
  - workflow
  - git
---

# Merge Then Update Consumers

When shipping a shared utility, update all consumers in the same or immediately following commit. Don't let the old code linger.

## The Pattern

```
Commit 1: Merge the shared utility (vault-search script)
Commit 2: Refactor all consumers to use it (Librarian, Company Snapshot)
Commit 3: Build new features that depend on it (morning briefing vault context)
```

Not:
```
Commit 1: Merge the shared utility
... weeks pass ...
Some consumers still use the old approach
New consumers don't know the utility exists
```

## Why It Works

- Proves the utility actually works in real consumers, not just in isolation
- Prevents "we have a shared utility but nobody uses it" drift
- The refactoring commit is easy to review because the utility just merged — the reviewer has fresh context
- New consumers see the pattern and follow it

## When to Use

Every time you ship a shared utility, helper function, or config file. The utility isn't done until at least one consumer uses it. If you can't update a consumer in the same session, file an issue linking the utility to the consumer that needs updating.

## Source

Observed in a course participant's skills repo: vault-search merged, then Librarian + Company Snapshot refactored to use it in the very next commit, then morning briefing built on top of it. Three commits, clean dependency chain, no stale code left behind.
