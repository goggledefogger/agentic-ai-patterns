---
type: pattern
date: "2026-08-09"
source: The personal agent, 2026-08-05 — two diagnoses run against a working copy that was behind origin; one built a duplicate LaunchAgent against a roadmap row the remote had already superseded, the other spent three tool calls on a symlink that had been replaced minutes earlier
tags:
  - agent-safety
  - staleness
  - diagnosis
  - version-control
  - multi-session
---

## The Problem

An agent opens an investigation and starts reading: grep the repo, read the roadmap row, check the config, form a hypothesis. Every one of those reads is accurate about the tree in front of it. The tree is behind its remote.

Nothing announces this. The tools do not fail, the files are not corrupt, and the output has the same shape and the same confidence as a correct diagnosis. The agent then reports a problem that was fixed upstream, or builds a thing that already exists, and it does so with citations.

Two recorded bites, same week, same cause. One agent read a roadmap row saying a guard was unbuilt, and built it — the remote had shipped it, and the row it trusted predated the work. Another diagnosed a failing service three tool calls deep against a symlink that had been replaced minutes before the session started.

## The Pattern

Any session that will diagnose, report on, or build against repo-backed state opens with a fetch, **before the first read**, not after the answer starts looking strange:

```
git -C <repo> status --short     # whose work is in the tree?
git -C <repo> pull --ff-only     # then, and only then, start reading
```

Two bounds make it safe, and both are about other people's work:

1. **Status first, and do not pull into a tree someone else is mid-edit in.** Name the dirt and leave it. A dirty tree is a signal to proceed read-only, not an obstacle to clear.
2. **Pull at the start of the work, never in the middle.** A fast-forward landing underneath your own in-flight edits is a second bug, and a worse one, because now the corruption is yours.

## Why It Works

Staleness is invisible at the point of use. That is the whole difficulty: there is no read that reveals it, because every read is internally consistent. The only place the problem is visible is *before* any reading has happened, when the question "is this copy current?" is still separable from "what does this copy say?"

Moving the check to the top costs one second and converts an unfalsifiable class of error into a non-event. It also front-loads the concurrency question at the one moment when the answer is still cheap to act on.

## When to Use

- Before diagnosing anything in a repo-backed system
- Before reading a roadmap, decision doc, or status file to decide whether work is already done
- Before building against a checkout that a CI job, a teammate, or another agent session also writes to
- At the start of any long session in a shared tree

## When NOT to Use

- Mid-task, once you hold in-flight edits — finish or stash deliberately first
- In a tree another session is actively editing; read it as-is and say so
- Where the remote is not the source of truth for the question being asked (a local-only experiment, a deliberately pinned checkout)

## Watch-outs

- `--ff-only` is doing real work here. A merge or rebase mid-session is exactly the "second bug" the bound exists to prevent, so let the pull fail loudly rather than resolve anything.
- A clean `git status` is not proof the tree is current — it only means nothing local is uncommitted. Clean and stale is the common case.
- The failure recurs specifically around documents that describe state (roadmaps, status files, decision logs), because those are the files most likely to have been updated by the very work you are about to duplicate.

## Adjacent Patterns

- `stale-pointer-asserts-confidently` — the nearest neighbour and the general form: hold the question, resolve at use time, rather than caching an answer. A stale checkout is precisely a cached answer.
- `freshness-axis-must-match-the-question` — related but distinct. That pattern is about a drift checker's *precedence logic* being fooled when version-control operations rewrite mtimes; this one is about staleness silently *causing* a misdiagnosis. Both share the moral that clocks are a poor proxy for currency.
- `multi-session-fleet-awareness` — names the cost of concurrent work, which is what bound 1 is protecting.

## Source

The personal agent, 2026-08-05. Checked against the library before writing: no existing pattern documented pull timing, when not to auto-pull, or the concurrent-session fast-forward hazard.
