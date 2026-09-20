---
type: pattern
date: "2026-05-01"
source: A course participant's home-agent system (the workspace repo, PR #52)
tags:
  - git
  - workflow
  - automation
---

# Pre-Push Hook for PR Discipline

A 100-line `.githooks/pre-push` script that mechanically blocks direct pushes to `main` on a specified set of repos. Allows squash-merge subjects (`... (#NN)`), no-ff merges (`Merge pull request #NN ...`), and BUILD-STATUS-only commits. Bypass with `--no-verify`. Encodes PR discipline as code, not as memory.

## The Problem

CLAUDE.md says "use PRs." For solo work the temptation to push direct is high — it's faster, the change is small, no one's reviewing anyway. The cost lands later: a parallel branch grows out of date, a teammate joins and finds main commits that bypass review, a Claude Code session pushes "while I'm here" and breaks something nobody saw.

Documentation alone doesn't solve this. The discipline has to be mechanical, because the moment of temptation is the moment when "I'll just push this one" sounds reasonable.

## The Pattern

A pre-push hook with three components:

### 1. Repo allowlist

Only fire on repos that have committed to the PR workflow. Other repos pass through silently:

```bash
case "$url" in
  *org/repo-a*|*org/repo-b*) ;;
  *) exit 0 ;;
esac
```

### 2. Branch-target check

Only block pushes to `main` or `master`. Feature branches always pass:

```bash
case "$remote_ref" in
  refs/heads/main|refs/heads/master) ;;
  *) continue ;;
esac
```

### 3. Allowed-pattern allowlist

For each new commit being pushed to main, check if it matches an allowed pattern. Block otherwise:

- **Squash-merge subjects** matching `... (#NN)$` — the GitHub UI's default merge format
- **No-ff merge commits** matching `^Merge pull request #NN ...` — alternative GitHub merge format
- **Generated-artifact-only commits** that touch only `BUILD-STATUS.md` (or whatever your regenerated artifacts are)

Anything else gets blocked with a message naming the violating SHA and subject.

## Bypass

`git push --no-verify` exists for the rare legitimate case (urgent hotfix, intentional artifact commit, recovering from a hook bug). The bypass is loud — it's an explicit flag. CLAUDE.md should name when the bypass is appropriate.

## Installation

`scripts/install-hooks.sh` sets `core.hooksPath = .githooks`. Anyone who runs the install picks up every hook automatically. New hooks added later don't require manual sync. The Mini and any contributor who's installed hooks gets the discipline without coordination.

**The caveat that belongs in the installer, not in someone's memory:** `core.hooksPath` makes git ignore `.git/hooks` **entirely**. Any hook another tool dropped there — a formatter, a secret scanner, an IDE integration — stops firing, with no error, no warning, and no output. The repo that just gained a hook can silently lose one. So the installer should look before it sets:

```bash
shadowed="$(find "$git_dir/hooks" -maxdepth 1 -type f ! -name '*.sample')"
[ -n "$shadowed" ] && { echo "REFUSING: these would be silently ignored"; exit 1; }
```

Refuse and name them rather than shadowing them. The alternative install — symlinking each hook into `.git/hooks` — keeps other hooks working but loses the auto-pickup of new ones, so it trades the failure you can see for one you can't. Prefer `hooksPath` **with** the guard.

## Validate the commit being pushed, not the working tree

A hook that inspects the working tree answers a question nobody asked. What is about to become public is the pushed commit; the worktree is whatever happens to be lying around, including another session's untracked scratch directories.

This is not hypothetical where several agent sessions share a checkout. A validator run against the worktree flags a peer session's WIP and blocks an unrelated push — and a hook that blocks pushes for reasons the pusher did not cause is one that gets `--no-verify`'d permanently within a week. The bypass is not the failure; the hook teaching people to use the bypass is.

Check out the pushed SHA and run there:

```bash
git worktree add --detach --quiet "$tmp" "$sha"
trap "git worktree remove --force '$tmp'" RETURN
(cd "$tmp" && python3 scripts/validate.py)
```

Use a linked worktree, not `git archive | tar`: a tarball is not a git repo, so any validator that shells out to `git ls-files` (a common way to ask "what is tracked here") exits 128 and the hook fails for the wrong reason. That failure is loud and confusing rather than silent, but it still costs a debugging round.

Read the refs from stdin — pre-push receives `<local_ref> <local_sha> <remote_ref> <remote_sha>` per line — and skip deletions, whose local SHA is all zeroes.

## Forensic justification

The commit message that introduces the hook should be **forensic** — name the specific incidents that motivated it:

> *"Forensic check on 2026-04-30: 6 substantive direct pushes to main in the prior week, plus PR #43 stuck because ba4dbab created a parallel crm-updater/SKILL.md without going through review. The hook would have caught all 6."*

This matters because every future contributor (and future-you) needs to understand the hook isn't preventive hygiene — there were real failures, the hook closes a real gap. Without the forensic justification, someone bypasses it the first time it's inconvenient.

## When to Use

For any repo where:
- The PR workflow is policy, not preference
- More than one contributor (human or agent) pushes
- Regenerated artifacts (status files, build outputs, lock files) commit cleanly via direct push
- "Quick fix to main" has caused incidents

Skip for:
- Solo throwaway repos
- Repos where main is the working branch by design
- Repos where you genuinely want every push to land directly (release-tag-only repos, content-only repos)

## Example from the participant's system

`.githooks/pre-push` (PR #52, commit `eb1f2cd`). Fires on `the workspace repo`, `the sensor service`, `the memory pipeline`, `a private org/an airtable-mcp repo`, `a private org/carta-mcp`. Forensic commit message names 6 prior incidents the hook would have caught. Used `core.hooksPath = .githooks` for zero-friction install.

The hook explicitly handles fast-forward cases: when a PR is merged via the GitHub UI, the local main can fast-forward to a commit that already exists on origin/main. The hook fetches origin's view of main first and only checks commits that aren't already there. This means "merging a PR via GitHub then pushing local sync" passes cleanly — only genuinely-new direct work gets blocked.

## Adjacent Patterns

- **An unenforced red becomes the baseline** (`unenforced-red-becomes-the-baseline.md`) — why you reach for this hook at all. A CI check that refuses correctly but blocks nothing leaves the default branch red, after which every later run carries no information. This hook is the client-side binding you install when the server-side one (a required status check) needs a paid plan or a public repo.
- **Decision-doc ADR** (`decision-doc-adr.md`) — when you add the hook, write a decision doc capturing why. The hook is mechanical; the doc captures intent.
- **Forensic commit messages** — naming specific incidents in the commit body is a generalizable pattern. Applies to any rule-as-code change (lint rules, CI checks, test additions). Without the forensics, the rule looks like preventive hygiene and gets disabled the first time it's inconvenient.

## How to Adopt

1. Copy `.githooks/pre-push` from `the workspace repo` into your repo (or the obsidian-claude template repo as a default).
2. Edit the repo allowlist for your specific repos.
3. Edit the BUILD-STATUS-only allowlist for your generated artifacts (or remove if you have none).
4. Run `git config core.hooksPath .githooks` (or copy `scripts/install-hooks.sh` from the participant's system).
5. **Before committing the hook**, do the forensic check: `git log --first-parent main --since="2 weeks ago" --format="%H %s"` and identify which commits would have been blocked. Name them in the commit message.
6. Test the hook fires: try a direct push to main on a throwaway commit. Test the bypass works: `git push --no-verify`. Test the allowlist passes: merge a PR and push the resulting fast-forward.
