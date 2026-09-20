---
type: pattern
date: "2026-07-26"
source: The agent's own repo, backup-onedrive red for a week under launchd, scheduler PATH incident, 2026-07-26
tags:
  - debugging
  - memory
  - anti-pattern
  - monitoring
  - operations
---

# A Plausible Cause Ends the Search

An agent holding durable memory will explain a live failure before it reads one. The
explanation is usually *true* — just not the cause. And because it fits, nothing ever
contradicts it, so no amount of "trust the live source over stale memory" fires.

## The failure

The personal agent's session-start nudges flagged a scheduled backup task as failing. The personal agent held a memory,
written six days earlier and entirely accurate, saying that backup had been deliberately
paused by unplugging its external drive. The personal agent connected the two, called the alarm a false
red light, and proposed a plan built on that reading — without opening the log.

The log said `rclone is not installed or not in PATH`. rclone was installed. It was
invisible because launchd starts jobs with `PATH=/usr/bin:/bin:/usr/sbin:/sbin` and the
binary lives in a Homebrew prefix. And the unplugged-drive branch **exits 0 by design** —
a paused drive could never have turned anything red. The remembered cause was not merely
unproven, it was structurally incapable of producing the symptom.

Consequence: the offsite backup was broken, not resting. It would have stayed broken
through the replug that everyone believed was the fix. The memory was what kept it
comfortable.

## Why the usual guardrail misses it

Most "don't trust stale memory" rules are written for **contradiction**: memory says X,
the live source says not-X, live wins. That works because the conflict is visible and
trips the override.

This failure has no conflict. Memory offers a cause, the symptom is consistent with it,
and the search terminates satisfied. The agent never *disbelieves* the evidence — it
never reaches the evidence. Coherence is doing the damage that staleness usually gets
blamed for.

The subjective tell, worth teaching directly: **the feeling of having explained something
you have not looked at.** It is indistinguishable from understanding, from the inside.

## The rule

**A remembered cause is a hypothesis, and it does not get to outrank evidence it was never
checked against.**

Order is the whole fix:

1. Read the actual failure output first — log, exit code, stderr.
2. *Then* check whether memory matches it.
3. Never the reverse, and be most suspicious exactly when step 0 already handed you a
   satisfying story.

Note the rule stops short of "the error message is the truth," because in this very
incident it was not: `rclone is not installed` was **false as written** — rclone was
installed and merely unreachable. `notebooklm-research.md` already carries that half ("a
tool's error message is a hypothesis, not a diagnosis"), and the two bound each other
neatly. The error is where you are obliged to *start*; it is not where you are allowed to
*stop*. Memory doesn't even earn the start.

A corollary for anything that reports state: before accepting "this alarm is expected,"
confirm the expected condition can *actually produce* this signal. Here it could not —
the benign branch exits clean. One glance at the code would have killed the theory faster
than the log did.

## Adjacent

- `verification-needs-a-negative-control.md` — same family. There, a test that cannot fail
  proves nothing; here, an explanation that cannot be contradicted diagnoses nothing.
- `notebooklm-research.md` — the opposite guard, and the necessary complement: an error
  message is itself a claim to verify. Together: read the error, then check what it says
  is actually true.
- `system-understanding-protocol.md` — grounding claims in citations before asserting them.
- `unattended-run-discipline.md` — the environment half of this incident: a tool present in
  your shell and absent from the daemon's PATH. "It works when I run it" is not a check.

## Human note

The user broke the loop, and how they did it is the reusable part. Not a correction — a
refusal to accept a confident frame that had no evidence under it:

> "I'm not sure if they're lying or if they're actually true because I don't know... just
> talk me through why you think it's lying or double check to make sure."

"Lying" was the agent's word, not the data's. Asking where a confident word came from is
often cheaper than checking the claim itself, and it catches the class of error where the
agent has quietly substituted a story for a look.
