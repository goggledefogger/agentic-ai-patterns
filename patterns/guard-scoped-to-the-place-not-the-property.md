---
type: pattern
date: "2026-08-09"
source: The agent's own repo, a script and walk-and-talk player.html — two unrelated subsystems, same defect, found four hours apart in one session. Third instance 2026-08-23: The household-agent repo a script, where the fix for a place-scoped guard was itself place-scoped and shipped with a live instance of its own defect
tags:
  - guards
  - debugging
  - anti-pattern
  - staleness
---

# A Guard Scoped To The Place, Not The Property

Two guards, two subsystems that share no code, one night.

**The commit guard.** A pre-commit check refuses to let one session commit over
a path another session is actively editing. Each session appends `{ts, path}`
records to its own manifest file. The liveness test read the **file's mtime**:

```python
if not is_live(manifest_file, now, stale_secs):
    continue
```

**The speech de-duplicator.** A phone recognizer re-delivers one sentence
repeatedly as it revises it, and the fix for that is to re-glue an overlapping
tail instead of appending it. The re-glue was gated on a flag meaning *a
recognizer session was restarted with words still pending*:

```js
if (uttResumed) {
  // ...tail-overlap re-glue...
}
```

Both shipped working. Both were written against a real, reproduced failure. Both
were blind by construction to the next instance of the very thing they guard.

## What actually went wrong

Neither guard checked the property that made the case a case. Each checked a
**place where that property had been observed.**

The commit guard needs to know *is this claim still live*. It asked *is this
file recent*. Those agree right up until a session touches something unrelated —
then one mtime bump silently re-arms every stale claim that session ever made. A
peer claimed a file at 15:20, committed it at 15:21, touched a different file at
19:59, and the guard denied that path until 21:59. Six hours, on work that was
finished, by a session that was closed.

The de-duplicator needs to know *is this text a re-delivery*. It asked *did the
recognizer just restart*. Those agree in the case the bug report came from, and
diverge the moment the same re-delivery happens **inside** one live session,
where the flag is false by construction. One sentence arrived twelve fragments
deep: *"see let's / let's just / let's just to / let's just to your point / …"*

Same shape. The guard's condition names a **location, a phase, a boundary or a
mode** where the property was true, instead of naming the property.

## Why it keeps happening

Because a bug is discovered *somewhere*, and the place is the most vivid thing in
the report. "It happens after a restart." "It happens in that file." The fix gets
written against the report rather than against the mechanism, and it passes,
because the reproduction is the place.

The place is not a mistake at the time — it is genuinely correlated. It becomes
a mistake later, when the property shows up somewhere the place does not cover,
and the guard does not fail loudly. It simply does not fire.

That is the expensive part: **a place-scoped guard degrades into silence, not
into an error.** The commit guard did not crash, it denied. The de-duplicator did
not throw, it concatenated. Both looked like the system working.

## The Pattern

**When a guard's condition names *where* it may act, treat that as a code smell
and ask what that place guarantees. Check for the guarantee instead.**

- Commit guard: the file's recency was standing in for *the claim's age*. Age
  each record against its own `ts`.
- De-duplicator: the restart boundary was standing in for *arrival recency* — a
  genuine next sentence needs a pause first, a revision does not. Gate on the
  gap since the last update.

Both fixes made the guard **shorter**, not longer, and both removed a flag rather
than adding one. That is usually the sign you found the property rather than
another place.

## Keep the place as an optimization, if it is sound in one direction

Not every place-check is wrong to keep. Records are appended, so no record can be
newer than the file that holds it: a stale file therefore contains only stale
claims and can still be skipped whole. The converse never held, and that
asymmetry is the whole bug. So the mtime check survived — demoted, with the
demotion written into its docstring:

> This is a sound EARLY-OUT only, never the aging decision itself.

If you keep a place-check, say in the code which direction it is sound in.
Otherwise the next reader restores it to load-bearing.

## Writing the test so it cannot lie

A test built from the same reproduction inherits the same blind spot. Both fixes
needed a case where **only the property varies and the place is held constant**:

- Two claims in one manifest — one six hours old, one a minute old. The old one
  must not deny; the fresh one must. Same file, same peer, same everything but
  age.
- A re-delivery arriving inside a live session, where the restart flag is false.

Then run that test against the *pre-fix* code and confirm it fails. A dedup or
staleness test that passes on the broken version is describing the fix, not the
defect.

## The pattern recurs inside the fix, and the same tell catches it

Both cases above were fixed correctly on the first attempt. The more common
outcome is that **the remediation is itself place-scoped**, because the person
writing it has just finished studying one lane and that lane is now the vivid
thing.

A wiki write-guard validated that a capture marked `verbatim_capture: true` was
not secretly an agent's summary. It engaged only when the source string looked
like a podcast or a YouTube link — the lane where the defect had first been
reproduced. A dictated note therefore skipped the body read entirely and passed,
and a household knowledge domain got built on four bullet points the speaker had
never said. Classic place-for-property.

The fix added a second lane — dictation — to the allowlist. It shipped green,
with tests, and closed the reported case.

It was still the same defect. The census that should have been run first: of 64
captures asserting verbatim, the original guard read 23 and the fix took that to
29. **Thirty-five were still never read**, and one of them already carried the
exact shape the fix existed to catch, escaping only because its source string
matched neither lane. The new guard shipped with a live instance of its own
defect.

The tell was available before any of that measuring, in this pattern's own
diagnostic: **the fix made the guard longer and added two flags.** A property-fix
usually makes it shorter. When your remediation for a place-scoped guard grows
the condition rather than replacing it, you have found another place.

The property-scoped version asked what every such claim actually asserts — *the
body holds the source's own words* — which is checkable without knowing the lane.
The lane test disappeared, coverage went to all 64, and the checks that stayed
scoped had a reason written next to them rather than an inherited one.

### The carve-out is where the place sneaks back in

Property-scoping widens what gets examined, so it needs exclusions — and an
exclusion is a place-check wearing a different hat unless it names a property.

Twice in one change a genuine capture got flagged: a spec sheet transcribed off a
photo, and a task envelope from an agent message bus. Both are label-per-field by
nature, so a bulleted body is faithful rather than a paraphrase. Both were
resolved the same way — **probe the source format, then name the property**
("labels *are* the content"), rather than adding the lane that happened to
produce the false positive.

Neither was caught by review. The first came from running the new validator
against the real corpus before deploying; the second from reading the message
bus's own format spec. That is the general move: **a guard's blast radius is not
derivable from its diff, only from its population.** Run the new predicate over
the live data and read every disagreement before shipping — the disagreements are
the design review.

## Related

- [[a-verdict-is-scoped-to-the-worker-that-reached-it]] — the sibling error:
  scoping something correctly to a place, then letting it escape that scope.
- [[unrun-checks-read-as-passing]] — the same silence-not-error failure mode,
  one layer up.
- [[an-event-is-not-a-cause]] — mistaking the circumstance for the mechanism.
