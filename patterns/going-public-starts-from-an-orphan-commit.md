---
type: pattern
date: "2026-10-07"
source: A repo with private history was made public, the history leaked for a minute, and the cleanup showed which remedies remove commits from a host and which only hide them
tags:
  - git
  - publishing
  - privacy
  - safety
---

# Going Public Starts From an Orphan Commit

A repo that grew up private carries its private past in every commit. Deleting the files today does not help, because the history still has them. The clean way to make it public is a new root: one orphan commit of the current tree, with the full history kept on a local branch that never leaves the machine.

## The Pattern

1. **Make the tree clean first.** The `.gitignore` already excludes the private files, and `git status` shows nothing private staged
2. **Keep the history locally.** `git branch backup-full-history`. This branch is never pushed
3. **Start an orphan branch and commit once.** `git checkout --orphan public`, `git add -A`, `git commit -m "first public commit"`
4. **Rename it to the public default.** `git branch -M main`
5. **Verify, and read the output.** `git branch --show-current` says `main`. `git log --oneline` shows exactly 1 commit. `git ls-files | grep -E 'raw|private|drop'` (your own private patterns) returns nothing
6. **Push as its own step.** A plain push. The branch does not exist on the remote yet, so no force is needed. If a gate refuses `-f`, that is a hint you were about to use more power than the job required

## When It Leaks Anyway

The order above went wrong once, and the full history was public for about a minute. The cleanup taught the part that is easy to get wrong.

A force push does not remove commits from a hosting service. Deleting the branch does not either. The old commits stay on the host and can be fetched by hash until the repo is deleted or the host garbage collects them. Anyone who copied a hash, and any cache that saw it, can still reach them. Rewriting what the branch points at only changes what the front page shows.

What works, in order:

1. **Make the repo private right away.** It stops new readers while you work
2. **Delete it and create it again.** A deleted repo takes its unreachable commits with it
3. **If deletion needs a permission you do not have,** rename the leaked repo (still private) out of the way and create a fresh repo under the intended name. Then push the orphan branch there. The leaked copy stays private until someone with the right scope deletes it

Treat anything that was in the leaked history as seen. Rotate any secret, and tell anyone whose material was in it.

## Why It Works

An orphan commit has no parents, so the public repo starts with nothing behind it to inspect. The private history still exists, which matters because it is the real record of how the thing was built. It just lives where only the owner can reach it. The verification step in the middle is what makes the route safe, since every earlier step can fail quietly.

## Watch-outs

- Pick the branch names by reading them. A repo whose default is `master` breaks any command that assumes `main`
- Build artifacts, screenshots and drop folders are part of the tree. Check them, not only the files you think of as data
- Pushing tags can carry old history with it. Push the branch only

## Adjacent Patterns

- [[a-publish-step-gets-its-own-command-line]] is how this route went wrong: the setup and the push shared one shell line
- [[a-derived-public-dataset-leaks-through-its-provenance]] is the sweep to run on the tree before step 3
- [[ship-the-public-catalogue-with-a-private-overlay-beside-it]] is where the excluded files go so the owner's copy stays whole

## Source

A catalogue repo made public on 2026-10-07. A chained command published the full history for about a minute. Making it private, then moving it aside and creating a fresh repo under the same name, was the remedy that actually removed the commits from view.
