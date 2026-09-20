---
type: pattern
date: "2026-08-01"
source: The personal agent — walk-and-talk shipped "the synth dies with the session" and verified it, while nothing anywhere ever started the synth; found 2026-08-01
tags:
  - lifecycle
  - verification
  - acceptance-criteria
  - anti-pattern
---

# The Teardown Was Verified. The Startup Was Never Written.

A story says: *"it dies cleanly with the session — no orphan holding 2.4 GB after a walk ends."* Someone builds it, tests it by killing the session, watches the process disappear, and marks the criterion met. It **is** met. The teardown works.

Nobody asks the question the criterion quietly implies: does the thing ever *start*?

## The Problem

`walk-and-talk` had a resident speech server. Story 2.1 AC3 required it to die with the tmux session. The fix landed, was verified by exercising — kill the session, confirm the port is free — and the acceptance criterion was honestly satisfied.

Grepping the entire skill for anything that *launched* that server returned exactly one category of hit: the cleanup hook that kills it.

```sh
# walk-tmux.sh — the whole of the server's lifecycle, as shipped
tmux set-hook -g session-closed "run-shell 'lsof -ti tcp:$PORT ... | xargs kill -9'"
```

On every real walk the server was never running, so the phone fell through to a slow path and used the wrong voice. Meanwhile the planning doc contained this sentence, written before the fix:

> *"Epic 1 would serve audio from a process that **may not be running**."*

The author noticed the risk, wrote it down, built the teardown, and never noticed that "may not be running" was actually "is never running." The word *may* did the damage: it framed absence as an edge case, when absence was the default and the only state.

Two properties made it durable:

- **The teardown test passes identically whether or not startup exists.** Killing a session with no server running frees the port just fine. The verification was real and proved nothing about the half that mattered.
- **The fallback worked.** The system degraded to a lesser path instead of erroring, so there was no failure to investigate — only a quality complaint ("the voice is wrong") that pointed at the voice rather than the lifecycle.

Later, a second implementation of the startup made the mirror-image mistake: it added a start block and did **not** add the matching stop to the shared `cleanup_all` funnel — the same file whose comment reads *"CLEAN UP EVERYTHING THE SKILL STARTS."* A birth wired without its death, immediately after a death wired without its birth.

## The Pattern

**A lifecycle has two ends. Changing one is a change to both, and the test for either must distinguish "correct" from "absent."**

When a criterion mentions one end of a lifecycle, write down the other end explicitly before building:

| The story says | Also answer, in writing |
|---|---|
| "dies with the session" | what starts it, from which entry points, and does every entry point reach that? |
| "started at boot" | what stops it, and what happens on a second boot with the first still running? |
| "cleaned up on exit" | is there an exit path that skips the cleanup (crash, kill -9, a second entry door)? |

And the test that actually catches it:

```
1. From nothing, run the real entry point.     → assert the thing EXISTS
2. Exercise it.                                 → assert it does the work
3. Run the real exit path.                      → assert it is GONE
```

Step 1 is the one that was missing, and it is missing because it feels too obvious to write. Step 2 is what distinguishes this from a port check — a process can exist and be useless (see [[health-check-that-never-exercises]]).

## The Structural Fix

When more than one door leads into the same lifecycle, the start belongs in **one script both doors call**, not inlined in whichever door was being edited at the time:

```sh
# kokoro-up.sh — one birth, callable from every door
lsof -ti tcp:"$PORT" -sTCP:LISTEN >/dev/null 2>&1 && { echo "already up"; exit 0; }
[ -x "$VENV/bin/python" ] || { echo "no venv — degrading" >&2; exit 1; }
VIRTUAL_ENV="$VENV" nohup "$VENV/bin/python" "$HERE/server.py" --port "$PORT" --warm >"$LOG" 2>&1 &
```

Inlining it in one entry point is how the second door silently gets nothing — which is exactly what happened when the startup lived only in `walk-tmux.sh` and every session started via `session.sh` got no synth at all.

## It Recurred the Same Day

Hours after the synth's missing birth was fixed, the same walk stack failed the
same way one layer up: `session.sh stop` killed the phone **bridge** by design
(`cleanup_all`, whose comment says *"CLEAN UP EVERYTHING THE SKILL STARTS"*), and
nothing in `start` restarted it. The next session's phone page reconnected forever
against a dead port while replies were spoken aloud on the wrong machine. Same
class, second component, found within hours — which is the evidence that this is
structural, not a one-off oversight: **any funnel that collects deaths implies a
funnel that owes the matching births.** The fix was the same shape both times: a
small idempotent `*-up.sh` called from the start path, mirroring the entry in the
cleanup funnel.

## Related

- [[health-check-that-never-exercises]] — the thing exists but cannot work
- [[unrun-checks-read-as-passing]] — a check that never ran, scored as green
- [[decorative-gate]] — a guard that cannot see what it guards

## The Rule of Thumb

If a graceful fallback exists, an absent component produces no error — only a quality complaint pointing at the wrong subsystem. So when someone reports that a feature is "worse than expected" rather than broken, check first whether the component that makes it good is running at all.
