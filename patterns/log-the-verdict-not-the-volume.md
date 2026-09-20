---
type: pattern
date: "2026-08-01"
source: The personal agent — walk-and-talk, a phone on a walk has no console; three plausible-but-wrong theories were fixed before a logged HTTP status revealed the real cause, 2026-08-01
tags:
  - observability
  - logging
  - debugging
  - verification
---

# Log the Verdict, Not the Volume

The instinct when something fails invisibly is to add logging, which usually means adding *prints*. More lines, more detail, more places. Volume is not the property that makes a log useful. A log is useful when it can answer, without interpretation, the one question you will actually ask it:

> **Did the thing work, and if not, why?**

A log that requires you to reconstruct that answer from timestamps and prose has failed, however many lines it emits.

## The Problem

`walk-and-talk` renders speech on a Mac and plays it in a phone browser. When the phone used the wrong voice there was no console to read — the user is outdoors with the device in a pocket. Three plausible theories were diagnosed and fixed, in order, and **none was the cause**:

1. The audio element was never unlocked by a user gesture
2. A slow-starting media stream was being abandoned
3. The client timeout was shorter than the server's render time

The actual cause was that the page requested `/utt.wav` with a **truncated auth token** and got `403` on every single line. It hid because the browser translates a failed media load into its own vocabulary: an `<audio>` element pointed at a 403 fires `MEDIA_ERR_SRC_NOT_SUPPORTED` (code 4) — a code that reads like a codec problem and sends you looking at audio formats.

**An opaque error code is a missing boundary.** The fix that exposed the truth was not more logging; it was fetching the resource explicitly so the real transport status was in hand before the subsystem could translate it:

```js
fetch(url)
  .then(r => { if (!r.ok) throw new Error("http-" + r.status); return r.blob(); })
  .then(b => { audio.src = URL.createObjectURL(b); return audio.play(); })
  .catch(e => fallback("fetch:" + e.message));   // -> "fetch:http-403"
```

`audio-error:4` sent three people to the wrong subsystem. `fetch:http-403` named the bug on sight.

The second failure was in the log's *shape*. Two sides wrote to one file in two schemas — the page emitted `{ts, why, detail}`, the server `{ts, ev, id, status, ms}` — with no `side` field, no date, and **no join key**. It was verbose and still could not answer the question, because no line could be attributed to a particular utterance.

## The Pattern

**Design the log around the question, then write the reader that answers it.**

**1. One schema, both sides, one file.** Every participant writes the same envelope so the records are joinable:

```json
{"at": "2026-08-01T10:44:32", "side": "bridge", "ev": "utt", "id": 3, "status": 200, "ms": 5754}
{"at": "2026-08-01T10:44:32", "side": "page",   "ev": "kokoro-playing", "id": 3}
```

**2. A join key on every event.** `id` is what turns two streams into one story. An event without one cannot be attributed and is, for diagnostic purposes, noise.

**3. Get the real status before a subsystem translates it.** Whenever a component reports failures in its own vocabulary — media errors, ORM exceptions, driver codes — capture the transport status at the boundary yourself.

**4. Ship the reader with the log.** Raw JSONL is not observability; it is raw material. The reader states the verdict:

```
    id  when        render      size  outcome
     1  10:38:13    8029ms    201 KB  FELL BACK  audio-error:4
     2  10:37:33         —         —  FELL BACK  fetch:http-403
     3  10:44:32    5754ms    955 KB  GEORGE

  2 of 4 utterances used the chosen voice.
  why it fell back:  2 x fetch:http-403   1 x audio-error:4
```

**5. Never fold "no verdict recorded" into "failed."** On its first run the reader above reported *"0 of 3 used the chosen voice"* for a session where the right voice had demonstrably played — older events lacked the join key, so the verdicts were unattributable. It reported an absence of evidence as evidence of failure. Unknown is a third state and must print as one, or the log starts lying with a straight face while looking rigorous.

## Bracket the Lifecycle — a Dead Component Writes Nothing

The work log above has a blind spot: a process that is *gone* emits no events, so
its absence is invisible in its own log. Two cheap additions close it:

- **Birth and death certificates.** Log one line at boot (`{ev:"boot", pid, port}`)
  and trap SIGTERM/SIGINT to log one at shutdown with the reason. A shutdown line
  with no later boot line **is** the dead server a client was reconnecting to;
  a boot after a shutdown is a restart, timestamped. Without these, "the page said
  Reconnecting…" has no server-side story at all.
- **One line per client connect, none per drop.** A healthy page connects once, so
  *repeated* connect lines are the user-visible "Reconnecting…" rendered
  server-side — no per-drop bookkeeping needed to tell that story.

Same session, the same principle caught a fourth defect from the other direction:
a 15 s give-up timer kept ticking while the page was hidden — where the audio it
was judging was not allowed to play — and fired 55 s later against a clean render.
**A timeout must share the runtime of the race it judges**: pause the clock when
the platform pauses the contestant, or the timeout measures the suspension, not
the work.

## The Test

Before adding a log line, ask what question it answers, then ask whether a reader could answer that question from the file alone, mechanically. If answering still needs a human to correlate timestamps by eye, add the join key instead of the line.

## Related

- [[unrun-checks-read-as-passing]] — absence of evidence scored as a pass
- [[health-check-that-never-exercises]] — a green signal from a process that cannot work
- [[count-the-source-not-the-survivors]] — a correct count of the wrong population

## The Rule of Thumb

If the log cannot tell you which of two subsystems to open, it is not observability, it is exhaust. And when a system already degrades gracefully, the log is the *only* witness — a silent fallback leaves no error to find, only a quality complaint pointing somewhere unhelpful.
