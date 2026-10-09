---
type: pattern
date: "2026-10-09"
source: A roadmap file generated from a project board, with a check mode that re-reads the board; generate then check, back to back, reported the fresh file as stale
tags:
  - code
  - verification
  - anti-pattern
  - generated-files
---

# A Generator Checks Against the Read It Wrote From

A roadmap file is generated from a project board. The generator has a `--check` mode: re-read the board, regenerate in memory, compare with the file on disk, fail if they differ. Reasonable.

Then: run generate, run check, back to back. The check said the file just written was stale. Nothing was wrong with the file. Between the two reads, another session had closed an issue on the board. The check was not comparing the file to its source. It was comparing two moments.

## Why the wrong move was attractive

Re-reading the source is the obvious way to ask "is this still true?" That answers "has the board moved?" but it was used to prove the generator's own output, and for that a second read is a different world. The result is a flake that looks like a real warning, and people learn to ignore it or regenerate in a loop.

## The rule

**A generated artifact's freshness check must compare against the same snapshot the generator read, or the artifact must carry the snapshot's identity so the check can say "current as of X" instead of guessing.**

Carry an etag, a commit, a revision number or a timestamp inside the file. Then the check has three honest answers:

1. The file matches the snapshot it names: the generator is deterministic.
2. The source has a newer snapshot than the one named: the file is behind, and by how much.
3. Neither: something is broken.

Without a snapshot id, "stale" mixes the generator being wrong with the world moving on, and the output cannot tell them apart.

## The tell

**"Regenerate, then check, then it says stale again."** If a fresh write fails its own check, suspect the comparison before the file.

## A related move that was right

The generator also merges two sides: two people's status comments on the same item. When they disagree, it does not pick one. It reports a blind spot. A merge that surfaces its own disagreement is doing the job a freshness check should do for time: saying what it cannot know instead of guessing.

## Adjacent Patterns

- [A Generated File Accepts the Write It Will Not Keep](a-generated-file-accepts-the-write-it-will-not-keep.md): the other half of the generated-file life cycle.
- [A Count Inherits the Freshness of the Replica It Was Counted From](replica-freshness-travels-with-the-count.md): freshness has to travel with the value.
