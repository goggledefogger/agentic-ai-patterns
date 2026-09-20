---
type: pattern
date: "2026-08-24"
source: The personal agent's skill-listing budget — a SessionStart hook bulk-created 26 symlinks that blew a curated 8-skill route, found while investigating a five-day-old RED on the cloud-VM leader, 2026-08-23/24
tags:
  - architecture
  - reliability
  - anti-pattern
  - automation
  - debugging
---

# Disarm the Producer Before Deleting the Artifact

A `SessionStart` hook in the personal agent's `.claude/settings.json` linked every skill in the repo into the user-scope `~/.claude/skills`, guarded by `[ -e … ] || ln -s`. It had been there for seven weeks and read as inert the entire time, because it was: after the first run on a machine there is nothing left for it to do, so it changes no mtimes, writes no log line, and shows up in no diff.

On 2026-08-19 at 09:30:43 UTC the first session whose project directory was that repo started on a headless box, and the hook created 26 symlinks in a single second. The box ran a deliberately curated 8-skill walk lane with a measured 8,000-character listing cap; the listing went to roughly 11,500 and descriptions began being silently truncated. That state held for five days.

The obvious remedy was to delete the symlinks. It would have been wrong in a way that leaves no trace: **the deletion is what re-arms the hook.** A producer that writes only when its output is absent is dormant exactly as long as the output exists. Remove the output and the next ordinary event — here, someone opening a session, which is the next keystroke — puts all 26 back within a second, and the cleanup reads as having silently failed.

## The Pattern

**An idempotent producer is invisible in precisely the state you find it in, and deleting its output is the one action that wakes it.** Its success is the camouflage:

- Re-running it changes nothing, so a "did anything change?" check answers *no*, correctly, forever.
- The artifact is the only surviving evidence the producer exists.
- Therefore removing the artifact removes the evidence and arms the mechanism in the same stroke.

So the order is not a preference, it is the whole pattern: **disarm the producer, verify the disarm, then delete.** Cleanup performed in the other order is not a smaller version of the fix — it is a no-op that looks like a fix, and the next occurrence gets attributed to a fresh cause because the real one has already been ruled out by inspection.

The repeat interval sets the deadline. A daily cron gives you a day of false success; an event-driven producer gives you no window at all.

## The Corollary That Cost the Most Time

The investigation opened with a confident, wrong attribution: a documented CLI flag (`onboard_host.py --wire-skills`) that produces a byte-identical result and is the only *code* path that does. It was cleared as the cause only after a repo-wide search for symlink creation across the source directories came back with nothing else — a search that could not have found the real producer, which was a shell one-liner inside a settings file.

**A search scoped to where code lives cannot see a producer that lives in configuration.** Hooks, CI steps, unit files, launch agents, `postinstall` scripts and settings blobs all execute, and none of them are in `src/`. When a producer is unaccounted for, the search space is *everything that runs*, not everything that compiles.

## The Test That Finds It

Stop asking "what created this?" and ask **"what would create it again?"**

Take a scratch copy of the environment, delete one instance of the artifact, perform the single most ordinary triggering action once — start a session, push a commit, wait one tick — and look. If the artifact returns you have found the producer, and you have found it without needing to have grepped the right directory. If nothing returns, your inventory of producers is complete and the deletion is safe to perform for real.

This test is cheap, and it is specifically immune to the corollary above: it never asks where the producer lives, only whether one exists.

## Related

- [[a-trigger-must-not-watch-its-own-writes]] — the same "the cleanup is the trigger" shape from the write side; there a process re-arms itself by writing bookkeeping into the surface it watches, here a deletion re-arms a producer by restoring the absence it keys on
- [[a-generated-file-accepts-the-write-it-will-not-keep]] — the artifact does not disclose that something outranks it; that one is a renderer overwriting a hand edit, this one is a producer restoring a deletion
- [[always-loaded-context-budget]] — what the artifact actually cost here: headroom bought once and silently re-consumed, with nothing reporting the number until someone went looking
- [[half-gate-whole-verdict]] — the same failure of a compound claim, applied to cleanup: "the artifact is gone" is one conjunct of "this is fixed," and certifying the second from the first is how the loop closes
