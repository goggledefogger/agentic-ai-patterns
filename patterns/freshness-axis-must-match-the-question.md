---
type: pattern
date: "2026-08-02"
source: The agent's own repo, 2026-08-02 (pull direction) and the household-agent repo, 2026-08-12 (push direction). First: a drift checker compared repo and remote copies of a skill file by mtime, reported "Repo is NEWER", and skipped the pull — while the remote copy held content the repo did not. A git checkout the same morning had stamped a fresh mtime onto the older content, so the newer bytes lost to the newer clock. Second: a deploy gated on the same axis shipped a four-day-old branch over a merged fix on the host, exited 0, and left the content-hash drift check reporting clean afterwards.
tags:
  - verification
  - detectors
  - git
  - deployment
  - anti-pattern
---

# Freshness by Timestamp Answers a Different Question Than Freshness by Content

A `check-drift` tool compares a repo's copy of a file against the copy running on a remote host. It detects a difference, decides which side is newer, and pulls or pushes accordingly. On a file where the remote held real, wanted changes, it printed:

```
DRIFT: productivity/google-workspace/SKILL.md (Repo is NEWER)
```

and skipped it. The remote copy had content the repo did not. The "newer" repo file was **older content with a newer clock** — a `git checkout` that morning had rewritten the file's mtime while restoring bytes from a prior commit.

The tool detected the drift correctly. It resolved it backwards, and reported the backwards resolution as a decision rather than a problem.

## The Problem

Detecting *that* two copies differ and deciding *which one wins* are separate questions, and they want different evidence. Difference is a content question and is cheaply answered by a hash. Precedence is an intent question — *which copy holds the change someone meant to keep?* — and mtime is only a proxy for it.

Mtime is a proxy that breaks precisely when version control is involved, which is to say constantly:

- `git checkout`, `git stash pop`, `git restore` — all write current timestamps onto historical content.
- A fresh clone stamps every file with the clone time, making the entire history "newer" than any live edit anywhere.
- `rsync` without `-t`, container image builds, and archive extraction all reset mtimes.
- Restoring a backup makes the *oldest* content the newest by clock, at the exact moment you most need to reason about which copy is real.

The failure is quiet in the worst way: the guard runs, notices something, and announces a resolution. A skipped pull is indistinguishable in the output from a pull that had nothing to do. The operator sees a tool that ran and reported — not one that reached a wrong conclusion. In the case above, the drift was only caught because a human diffed the two copies by hand after the tool had already spoken.

This has a sibling in the same class: a deploy step that decides "already deployed" from a stale marker file. Same root — a *cheap proxy for identity* standing in for identity itself.

## The same axis error on the push side, where it destroys instead of skips

The pull direction skips a wanted change. The push direction overwrites one, and the harm is somebody else's merged work.

A deploy script guarded its writes with the mirror-image question: *is the remote copy newer by mtime, and different?* If yes, skip; otherwise ship. On 2026-08-12 a branch cut four days earlier was deployed twice while the default branch had moved. Every stale file carried a checkout-fresh mtime, so none of them looked older than the remote's correct copies, the gate cleared, and the deploy reverted a merged PR on the host — removing a mid-flight download size cap and a corrected process-kill target. Both runs exited 0 and reported nothing. It surfaced days later, only because a later merge produced a conflict a human had to read.

The gate did not fail. It was never able to see this: it was measuring when a file was touched, while the question was whether the branch being deployed *contains* what is already live.

Two corollaries, both counter-intuitive, both worth stating out loud:

**The optimisation that makes the sync fast makes its failure surgical.** The same deploy had recently been sped up by skipping byte-identical files. That means it copies exactly the files that DIFFER — which, after a colleague's fix lands, are exactly their changes and nothing else. A dumb full-copy sync from a stale tree reverts everything and is obvious within minutes. A smart one reverts only the recent fixes and looks like a clean, fast deploy.

**Afterwards, the drift detector agrees with you.** Once the bad push completes, the remote matches the stale branch byte for byte, so the content hash comparison — the correct axis, the one this pattern argues for — reports no drift. It is answering honestly; it was only ever asked *does the remote match my working tree*, never *does my working tree contain the project*. A detector whose baseline is stale produces a confident all-clear over the exact damage it exists to find.

**The push-side fix is the same precedence question, asked before the write:** is the default branch an ancestor of HEAD? If not, this tree is missing work that is already live, and no per-file comparison can tell you that, because every per-file comparison uses this tree as its reference. One `git merge-base --is-ancestor` before the first remote contact, fail-closed, with an explicit override. And because the baseline is now known to be questionable, the drift report states the branch's own standing before it renders a verdict, rather than letting "no drift" imply more than it measured.

## The Fix

**Compare content to decide difference. Compare provenance to decide precedence. Never let a clock decide either.**

- **Difference:** hash both sides. `md5sum`/`sha256sum` over the two copies is definitive and costs nothing at these sizes.
- **Precedence:** ask a question that survives a checkout. Is the remote content reachable from any commit in the repo's history? If yes, the remote is a deployed ancestor and the repo wins. If no, the remote holds edits that exist nowhere in version control — the remote wins, and that is exactly the case a drift checker exists to catch.
- **When precedence is genuinely ambiguous, do not resolve it.** Print the diff and stop. A drift tool that refuses to guess is more useful than one that guesses silently; the operator can read a diff in seconds and knows things the tool does not.
- **Never let "newer" be the whole verdict in the output.** `Repo is NEWER` reads as settled. `Repo mtime is newer, but remote content is not in git history — NOT resolving, diff below` reads as what it is.

If mtime must be used as a fast pre-filter, treat a clean result as *"nothing to look at yet"* and never as *"resolved in favor of the newer clock."*

## Related

- `unrun-checks-read-as-passing.md` — the general failure of a guard's silence being read as an answer. Here the guard isn't silent; it's confidently wrong, which is worse.
- `count-the-source-not-the-survivors.md` — same instinct: measure the thing you actually care about, not the artifact that correlates with it.
- `opaque-write-needs-a-read-back.md` — a write you didn't verify by reading back is a write you're guessing about.
