---
type: pattern
date: "2026-09-19"
source: The dashboard mobile pass, 2026-09-19 — the owner asked three times in one evening why his phone did not have what his desk had (pinned rituals, the model chooser's state, "any other settings that should carry over?")
tags:
  - architecture
  - state
  - mobile
  - separation-of-concerns
---

# Settings Follow the Member, Not the Browser

A local web app over a member's second brain kept its small preferences where web apps keep them: `localStorage`. Pinned rituals, the light-or-dark choice, how much of the agent's working to show, whether the member had been greeted, which chooser folds were open. For its designed use, one person at one desk, that is invisible and correct. The first time the member opened the same brain from a phone, the phone had none of it. He read the empty pinned row as "the ribbon is gone" and the folded chooser as "it doesn't have my options." Nothing was gone. Every one of those values was a fact about *him* that the app had filed under *this browser*.

## The Pattern

- **Name the two concerns as two variables.** A *member preference* is about how the member wants the brain to behave, and follows them to every device: pins, theme when explicitly chosen, verbosity, chime and notification preferences, greeted, a preferred command, which folds they leave open. A *device fact* is about this machine and stays in its browser: which microphone, a per-monitor zoom nudge, a notification *permission* (the preference to chime is the member's; the permission to make a sound is the device's).
- **Classify every key in one table**, in the code and in the change description, with a reason each. Ambiguous keys follow the member unless they describe the device. A key nobody can classify is a design question, not a storage question.
- **One mechanism, in the member's files.** Member preferences live in the brain's own state file, next to everything else the app derives from the member's files, read and written through one route, and pushed to every open window so a change at the desk shows on the phone without a reload. Not a second store, not a sync layer.
- **Adopt once, never overwrite.** A brain that ran the old code holds its preferences in some browser's `localStorage`. On first load, if the state has no such field and the browser has a value, carry it over once and mark the field present. Presence, not length, is the marker, so an emptied list is never refilled and a phone that loads first with nothing does not block the desk's copy from carrying over. Leave the old key as the fallback copy. Prove it with a test whose fixture holds the *old* shape.
- **Writable from anywhere the member is.** A preference is not a security write. If the app gates state-changing routes to the desk (it should, when it has no login), preferences stay outside that gate, because setting one from the phone is the case that surfaced the split.

## Why It Works

- `localStorage` is per-origin *and* per-browser, which reads as per-member right up until the member has two devices. The fusion is a ceiling that never announces itself: every single-device user validates it daily.
- Putting preferences in the member's files keeps the app's own rule ("it renders the member's files, it never stores what they do not hold") true for the small things too, which is where drift starts.
- The adoption marker turns a migration into a one-time read with no flag day: old brains keep working, and the first load from the browser that holds the values is the migration.

## When to Use

- A member asks a second time why another device does not have something they set. The second ask is the design telling you which variable to split (the-agents-home-is-not-its-library.md has the same rule for folders).
- Before adding any new `localStorage` key: classify it first. If it is a member preference, it goes in the state file from day one and never needs adopting.
- Any app whose "settings" are really two populations wearing one storage API.

## Adjacent Patterns

- `the-agents-home-is-not-its-library.md` — the same split, one level up: where the process runs versus what it observes.
- `memory-substrate-selection.md` — why the member's files are the source of truth and a view regenerated from them cannot drift.
- `portable-engine-local-profile.md` — device facts belong in the device's environment, never in the shipped engine; the mirror image of this pattern.
