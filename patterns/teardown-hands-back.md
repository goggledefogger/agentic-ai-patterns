---
type: pattern
date: "2026-08-07"
source: The personal agent — walk-and-talk: a "clean up both halves" teardown correctly identified the tmux twin and the user's original session, then killed the original and left the twin running unwatched, twice in one day (ux-debt 82, fix commit 7d034d1 in a personal skills repo)
tags:
  - lifecycle
  - multi-session
  - agents
  - anti-pattern
---

# Teardown Hands Back to the Surface the User Opened

When a system spawns a disposable duplicate of a surface the user opened — a tmux twin of a live conversation, a mirror process, a shadow session — teardown has exactly one safe target. The half that dies is always the one the system made. Never the one the user opened. A teardown that gets the *identification* right and the *targeting* wrong looks, from the outside, exactly like a teardown that works — right up until the user's own session is the one that vanishes.

## The Problem

Walk-and-talk twins a live terminal session into tmux so a phone can inject voice into it ([[a-twin-inherits-the-conversation-not-the-channels]]). For the length of a walk, two live halves exist: the original the user opened, and the disposable twin the skill made to be phone-drivable. Leaving the twin running after the walk ends is the bug ux-debt 82 exists to fix — an abandoned duplicate, correctly diagnosed.

The teardown built for it does real identity work: it walks pid, start-time, and process ancestry to tell the two halves apart, and it gets that part right. Then it kills the one it decided was the twin. Twice in one day, it killed the original instead — ending the conversation the user was sitting in front of — and left the twin running in a tmux window nobody had open. From the user's chair: their session vanished, and nothing said why.

The second incident is the useful one. It landed on a corrected identity check and still killed the wrong half, which rules out "the check was buggy" as the whole story. The bug was in the shape: a teardown that decides who dies by pointing at a target, using logic external to that target, gets to be wrong in whichever direction the next edit doesn't happen to cover.

## The Pattern

Move the question. Not "which one should die" — decided from outside, by a process pointing at another process — but "**am I the one that should die**," decided by each half about itself.

- **The identity check stops selecting a target and starts answering one question**, asked by the code that is about to close its own process: is this me? A wrong-direction bug needs a direction to get wrong. A process that can only ever close itself has none.
- **The system-made half closes itself, last, detached** — after all cleanup output has actually landed, so it doesn't cut off its own log mid-sentence. Nothing external aims at it; it aims at itself.
- **Guard the self-close so it can only fire on the disposable half.** In walk-and-talk: close only if the live original still exists, this process is running inside tmux, and the session name matches the pattern the skill uses when it creates a twin. Fail any one of those and it does nothing. The safe failure mode for a self-close guard is "stay alive," never "guess."
- **The user-opened half carries no kill logic pointed at anything.** There's no code path in it that can end another session, so there's nothing in it to invert.

## Why the Old Shape Kept Failing

Getting the identity check right and the direction wrong isn't rare — it's the default outcome of separating "which is which" from "who dies." Every place the two get stitched together (a boolean read the wrong way, a variable named for what it holds rather than what it's for, an early return reordered by an unrelated edit later) flips the target while the identification still looks correct in isolation, on its own test. A self-close guard collapses the two questions into one: there's no target variable left to invert, because the only thing the check can produce is "yes, close" or "no, don't."

## Related

- [[a-twin-inherits-the-conversation-not-the-channels]] — the disposable duplicate this pattern tears down, found the same walk
- [[teardown-verified-startup-never-written]] — the other way a lifecycle's two ends drift apart: one end wired, the other never written
- [[convergent-standup-sloppy-quits]] — put the intelligence in start because stop is where attention has already left; this one is stop-side and puts the intelligence in "which half is asking," not in a better stop ritual
- [[announce-the-move-in-the-old-room]] — same walk, same family: the surface the user is actually looking at is the one every design decision has to protect

## Source

The personal agent, walk-and-talk skill, 2026-08-07 (ux-debt entry 82). A "clean up both halves" teardown identified the tmux twin and the user's original session correctly (pid + start-time + ancestry) and then killed the original, parking the twin in an unwatched window — twice in one day. Fixed in commit 7d034d1 (a personal skills repo) by making the system-made half self-close, guarded, instead of being targeted externally.
