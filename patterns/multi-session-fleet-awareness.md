---
type: pattern
date: 2026-07-30
source: The personal agent walk-and-talk session — managing a dozen concurrent Claude Code sessions by voice
tags: [multi-session, fleet, coordination, observability, router]
---

# Give the manager session a fleet view, not the fleet's contents

When many Claude Code sessions run at once across projects, the session asked to
manage them needs a cheap, honest snapshot of who's running, who's waiting, and
who's abandoned — without reading any sibling's transcript into its own context.

## The Problem

A user with 10+ open sessions across projects (some active, some paused
mid-question, some other instances of the same persona) wants one session to
help triage them — often eyes-free, where they can't glance at windows. The two
naive approaches both fail:

- **Guess from memory** — the manager hallucinates what its siblings are doing.
- **Read the transcripts** — a sibling transcript is megabytes of another
  project's contents, at that project's sensitivity tier. Pulling it in blows
  the context budget and leaks private material into a session that may be
  lower-tier.

There is also no exact mapping to lean on: a running `claude` process does
**not** hold its transcript file open (lsof-verified 2026-07-30), so you cannot
tie PID → session file directly.

## The Pattern

Build the fleet view from process metadata plus transcript *headlines*, never
transcript bodies:

1. **Enumerate processes** (`ps` for `claude` CLIs, `lsof -d cwd` for each
   one's working directory) and group by project.
2. **Approximate the mapping honestly**: for a project with N running
   processes, take the N most recently written transcripts in its
   `~/.claude/projects/<munged-path>/` folder. Say it's an approximation.
3. **Classify by write-recency**: transcript written in the last ~3 minutes →
   *active now* (hands off); older → *paused*, with the age.
4. **Extract one headline per session**: parse the transcript tail backwards
   for the last real user message (skip tool results and sidechains), truncate
   to ~70 chars. That headline is what the manager speaks or shows —
   "mid-thread about a meeting, quiet 9 minutes."
5. **Observe, never inject.** The manager surfaces the menu ("stuck waiting on
   you / actively working / done, closable") and the *user* switches to a
   session to act in it. Sending input into a sibling session is a different,
   stronger primitive with its own consent story.

## Why It Works

- Headlines are enough to triage: state + age + last ask answers "which of
  these needs me," which is the actual question.
- Write-recency is a reliable activity signal even though PID→file isn't
  exact — an actively-working session writes its transcript every few seconds.
- Keeping bodies out preserves both the context budget and the tier boundary:
  a 70-char user-authored headline is the same class of thing as a window
  title, not a document exfiltration.

## When to Use

- A "chief of staff" / router session asked to triage the user's open sessions.
- Eyes-free (voice) workflows where the user can't see their own windows.
- Before engaging a project at all — the fleet view is ambient-awareness writ
  large: it shows a sibling already owns the project you were about to touch.

## When NOT to Use

- To *coordinate work between* agents — that needs explicit shared state
  ([[coordinate-agents-through-shared-state]]), not passive observation.
- To recover what a sibling actually did — read its project's git log and
  artifacts (the durable record), not its transcript.

## Watch-outs

- **"Active" means writing, not working.** A session blocked on a permission
  prompt looks paused; one running a long build looks active. The headline
  plus age gives the user enough to judge, but don't over-claim.
- **The N-most-recent heuristic can mislabel** when sessions in one project
  start and stop quickly — present it as "recent sessions here," not "this
  process is this session."
- **Headlines can carry names.** A snippet from a private-tier project's
  session ("draft the offer for <candidate>") is brief but real content —
  treat the assembled fleet view at the tier of the most sensitive project it
  covers when sharing it onward.
- **Two instances of the same persona** (two copies) will both see the same
  nudges and fleet. First-writer owns a task; the fleet view is how the second
  instance notices the first and stands down.

## Prior Art (surveyed 2026-07-30)

The ecosystem converges on the same shape, which validates the triage model:

- [nicknisi/fleet](https://github.com/nicknisi/fleet) — tmux dashboard whose
  core move is sorting sessions into *needs you / working / ready / idle*,
  needs-you first. That four-state triage is the vocabulary to reuse.
- [mixpeek/amux](https://github.com/mixpeek/amux) — control plane for dozens
  of sessions from a web dashboard or phone; the remote/eyes-free end.
- [bjornjee/agent-dashboard](https://github.com/bjornjee/agent-dashboard) —
  reads live pane output via `tmux capture-pane` instead of transcripts: a
  sharper signal (catches a session blocked on a permission prompt) but only
  for sessions already living in tmux. Transcript-mtime is the fallback for
  plain-terminal sessions; capture-pane is the upgrade once the fleet is
  tmux-hosted.
- [claude_code_agent_farm](https://github.com/Dicklesworthstone/claude_code_agent_farm)
  — 20+ agents with lock-based coordination; the "when observation isn't
  enough" end of the spectrum.
- Practice consensus ([GitButler](https://blog.gitbutler.com/parallel-claude-code),
  [Tactic Remote](https://tacticremote.com/blog/2026-02-28-managing-multiple-claude-code-sessions/)):
  one session per project, worktrees for parallel work in one repo, and ~4
  concurrent sessions as the human attention ceiling — a fleet view mostly
  exists because people exceed it.

## Adjacent Patterns

- [[coordinate-agents-through-shared-state]] — when observation isn't enough
  and agents must hand work to each other.
- [[parallel-branch-burst-conflict-debt]] — the git-level cost of parallel
  sessions that don't know about each other.
- [[anchor-rituals-to-session-start]] — the session-start hook is where a
  fleet check naturally runs.

## Source

The personal agent, 2026-07-30: a voice-first walk-and-talk session asked to "manage all the
other sessions." Implementation: `scripts/fleet_status.py` in the agent's own repo
(process sweep + transcript-tail headlines, `--json` for machine use).
