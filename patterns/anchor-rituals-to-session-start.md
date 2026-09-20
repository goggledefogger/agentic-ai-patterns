---
type: pattern
date: "2026-07-29"
source: The work vault — /ready-to-wrap and /close-day routinely skipped while SessionStart-wired checks fired every time; residue hook added 2026-07-29
tags:
  - adoption
  - hooks
  - discipline
  - meta
---

# Anchor Hygiene Rituals to Session Start, Not Session End

Wrap-up rituals — commit the work, refresh the status doc, file the loose ends — get attached to the end of a session because that is where they logically belong. They then get skipped, because sessions do not end, they stop. The reliable anchor is the *start* of the next session, which always fires and which the user always reaches.

## The Problem

Session-end rituals fail for a structural reason, not a discipline one: **there is no moment that reliably signals "end."** A session finishes when the laptop closes, when a meeting starts, when attention moves. There is no cue, so a command that must be typed at the end competes with the thing that pulled attention away — and loses.

Session *start* is the opposite. It is unambiguous, it always happens, and the user is oriented toward the project rather than away from it.

The evidence in one vault, split cleanly along that line:

**Fired reliably (all wired to something automatic):**

- SessionStart hook warning that `origin/main` is ahead — fires every session
- A `UserPromptSubmit` hook nudging a routing skill when external content appears — fired unprompted during the session that produced this pattern
- Pre-write lint and pronoun-check hooks
- Coaching-note *production*, wired into a transcript skill: **194 produced, none missed**

**Skipped routinely (all requiring the user to remember at the end):**

- `/ready-to-wrap` and `/close-day`, both well-designed, both forgotten most days. The user's own words: *"i forget to do ready-to-wrap and close-day most times"*
- Coaching-note *reading and deletion* — the documented lifecycle, never once run across 194 notes
- Status-doc refreshes, drifting between forced updates

Same vault, same person, same quality of artifact. The only variable is whether something fired it.

## The Pattern

**Move the wrap to the start of the next session, and make it detect residue rather than ask for discipline.**

A session-start check reads cheap filesystem and git signals for evidence that a previous session did not wrap:

- Uncommitted changes **plus no commit in N hours** — dirty alone is normal mid-work, dirty *and* stale is residue
- Ephemeral artifacts (briefs, coaching notes, scratch files) past their documented lifecycle
- Living docs whose "last refreshed" date has aged out
- Queues whose open count has grown

Three design rules make it survivable.

### Quiet by default, fire on change

Print only when the picture **changes** from the last print, with a long re-nudge interval (~3 days) while residue persists. A fingerprint of the current findings in a state file is enough. A check that prints the same three lines every session trains the user to ignore the region of the screen where it prints — see `quiet-by-default-notifications.md`.

### Guard every signal against normal working states

The naive "dirty tree means unwrapped session" check fires constantly in a vault with concurrent sessions, where the tree is dirty all day by design. The 12-hour-since-last-commit guard is what separates residue from work in progress. **Every signal needs the equivalent question: what normal state would trip this, and what second condition excludes it?**

### Report, do not repair

The check surfaces counts and leaves the fixing to the conversation. Auto-deleting stale files or auto-committing residue turns a nudge into an actor with opinions about work it cannot see the context for.

## What This Does Not Fix

The check makes debt **visible**, it does not pay it. A vault with 194 unread notes will report 194 every session until someone sweeps them. That is the correct behavior — the alternative is a threshold tuned high enough to stay quiet, which is the same as not checking. Expect the first weeks to surface an embarrassing backlog, and treat clearing it as its own scheduled job.

It also does not replace the end-of-session commands. They remain useful when the user does remember, and the session-start check is what catches the days they don't. Keep both; just stop letting the wrap depend on the one that fails.

## Verification

Test the state machine directly, not just the exit code:

1. With residue present and no state file: **prints**
2. Immediately again, unchanged: **silent**
3. After clearing all residue: **silent, and the state file is removed**

Run the exact wrapped form that will sit in the settings file (including any `2>/dev/null || true`), because a hook that fails silently is indistinguishable from one that has nothing to say — the `unrun-checks-read-as-passing.md` trap.

## Related

- `wire-into-existing-flows.md` — the parent pattern. This is the corollary: not every forcing function is equal, and session-start beats session-end for anything the user must otherwise remember
- `quiet-by-default-notifications.md` — the change-detection and cooldown discipline that keeps the check from becoming noise
- `self-reporting-staleness-check.md` — the scheduled-job altitude above this one, for corpus drift rather than session residue
- `unrun-checks-read-as-passing.md` — why the hook must be proven to fire
- `morning-briefing-pipeline.md` — the push variant, when the signal should reach the user before they open the project at all
