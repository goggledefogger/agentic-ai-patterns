---
type: pattern
date: "2026-07-21"
source: The personal agent PRs #32→#33→#41 (the agent's ops playbook chain), 2026-07-21 consolidation — #33 auto-closed mid-merge
tags:
  - git
  - github
  - workflow
---

# Stacked PRs: Retarget Dependents Before Deleting a Merged Base

Merging the bottom of a PR stack with `--delete-branch` closed the next PR in the chain — GitHub cannot keep a PR open whose base branch no longer exists, and a closed-for-missing-base PR **cannot be reopened** (the reopen API fails). The dependent's content had to come back as a brand-new PR, orphaning the original's review history.

The trap is invisible until it fires: `gh pr merge N --delete-branch` is the obviously-right habit for solo branches, and the deletion happens *before* you look at what else pointed at that branch.

## The Pattern

For a stack A ← B ← C (each PR based on the previous branch), merging bottom-up:

1. **Before merging A:** retarget B's base to the trunk — `gh pr edit B --base main`. B's diff temporarily shows A's commits too; that's cosmetic and disappears when A merges.
2. Merge A (delete its branch freely — nothing points at it anymore).
3. Repeat down the stack: retarget C to main before deleting B's branch.
4. After each merge, give the forge a beat to recompute (`mergeable` reads `UNKNOWN` for a few seconds), then merge the next.

One command order — retarget, then delete — is the whole pattern. If the base branch is already gone: recreate the PR from the still-existing head branch against main, link the dead PR in the body for the review trail, and merge that.

Related failure in the same family: two sibling branches (not stacked) that both changed one function needed a manual union merge — see `parallel-branch-burst-conflict-debt.md`.
