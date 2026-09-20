---
type: pattern
date: "2026-07-31"
source: Live walk-and-talk field session, 2026-07-31 — an ambient-activity checker reported 8 running agent sessions, zero of which were sessions, then stood down from writing for a colleague that was itself
tags:
  - monitoring
  - anti-pattern
  - agents
  - process-matching
---

# A Monitor That Can't Exclude Its Own Exhaust Will Eventually Tell You to Stop Working

An ambient-activity checker reported 8 agent sessions running. Zero of the 8 were sessions — its `pgrep` pattern matched the whole command line, *including environment variables*, so every unrelated subprocess carrying the tool's config path in its own environment counted as a hit.

The same script then reported "active work in progress" and made the agent stand down from writing, because the two recently-changed files it saw had been written by the agent's own session-start automation. It stood down for a colleague that was itself.

## The Pattern

A signal that cannot exclude its own exhaust, or that matches on a string its own environment contains, will eventually tell you to stop working.

1. **Match on identity, never on a substring of context.** A process check keys on process name / PID / a narrow argv pattern, not a `pgrep` match against the full command line — the full command line includes env vars, and any config path your own tooling sets is a string your own tooling will then match on.
2. **Exclude your own artifacts explicitly.** If a monitor's job is to detect *other* activity, it needs a positive list of what "self" looks like (its own PID, its own write timestamps, its own file paths) so it can subtract them before reporting. Don't rely on the absence of a coincidence — the coincidence is the failure mode, not the exception to it.
3. **A "someone else is working" signal needs a negative control.** Run the check with the monitor's own process paused and confirm the count drops to what's actually out there. If pausing your own process doesn't change the reported count, the count was never measuring other activity.

## Why It Bites Specifically

- **It reads as correct on the way in.** The `pgrep` pattern is short, works in the terminal, and returns *some* PIDs — nothing about the eight results looks wrong until you check what they actually are.
- **The two failures compound instead of canceling.** The inflated count (8 sessions that don't exist) and the self-attribution (files it wrote itself, read as someone else's work) point the same direction: the monitor sees more activity than exists, in both dimensions, because both bugs share the same root — the monitor has no model of itself as a participant in the system it's watching.
- **The failure mode is "stand down," which looks responsible.** A monitor that overcounts *other* work and then defers to it doesn't look broken. It looks polite. That's what makes it dangerous — it fails toward an outcome nobody will question.

## Watch-outs

- **This isn't unique to `pgrep`.** Any check that string-matches over a broad surface (full command lines, full file contents, an entire log line) rather than a narrow identity field (PID, process name, a tagged writer field) inherits this. If the broad surface can contain a string your own tooling produces, it eventually will.
- **Recently-changed-files checks need a writer, not just a timestamp.** "Was this file touched recently" can't distinguish a colleague's edit from your own session-start hook. If the mtime is the only signal, tag the write with who made it (a commit author, a process-tagged log line) so the check can filter by identity instead of guessing from recency alone.

## Adjacent Patterns

- `count-the-source-not-the-survivors.md` — a different shape of the same family: a monitored population downstream of the thing being measured for is the wrong population to count. Here the monitor is downstream of *itself*.
- `verification-needs-a-negative-control.md` — the fix in both cases is the same discipline: prove the check can report zero, by removing the thing it's supposedly detecting and confirming the number drops.
- `unrun-checks-read-as-passing.md` — same family of "absence of a complaint read as evidence of health," here inverted into "presence of noise read as evidence of other people's work."

## Source

Live walk-and-talk field session, 2026-07-31.

## Also seen: 2026-09-09, a watcher that could never exit

A loop written `while pgrep -f "rclone copy gdrive:Takeout"; do ...; done` carried that exact string in its own `argv` and so matched **itself** — it could not terminate, and every status check asked "is the download running?" got a permanent yes long after the transfer finished. Worse, a `kill -STOP` aimed at the transfer suspended the watcher instead, so the real job kept running while the machine reported it paused. Same root as the 2026-07-31 case, one layer in: the process was not just failing to exclude its own exhaust, it *was* the exhaust. Fix as stated here — key on process name (`pgrep -x`) or filter by `ps -o comm=`, and signal exact PIDs you have just verified rather than pattern-killing a string you also wrote into a script.
