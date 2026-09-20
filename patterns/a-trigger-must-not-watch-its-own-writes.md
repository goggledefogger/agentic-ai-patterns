---
type: pattern
date: "2026-08-05"
source: The personal agent's local transcription pipeline — a macOS launchd WatchPaths agent re-triggered itself on its own failure markers, found while dogfooding a two-machine setup, 2026-08-05
tags:
  - architecture
  - reliability
  - anti-pattern
  - automation
  - event-driven
---

# A Trigger Must Not Watch Its Own Writes

A macOS launchd `WatchPaths` agent watched a directory of audio files and transcribed new arrivals. On every failed attempt the scanner wrote a small failure marker — attempt count plus reason — beside the audio, inside the same watched directory. A file that fails deterministically (silence, no speech) therefore re-arms the trigger that just failed on it, forever.

Measured over 16 minutes: 145 attempts, 144 failures, one speech-model invocation per attempt, throttled only by launchd's ~10s floor between `WatchPaths` events, and 21 sync-conflict copies of the marker manufactured across two machines sharing the folder.

## The Pattern

**A process fired by changes to a surface must never write its own bookkeeping to that surface.** Any write into the watched location re-arms the watcher that just ran, whether or not the write looks like real output:

- A failure marker re-arms it.
- A lock file re-arms it.
- A temp file re-arms it.
- A refreshed "last tried" timestamp re-arms it.

None of these are the output the trigger exists to produce, and all of them are indistinguishable, to a watcher keyed on "something in this directory changed," from a new arrival. **The fix that matters is not the backoff duration, it's that a skip performs zero writes to the watched surface.** Bookkeeping belongs beside the surface, never inside it — a sibling directory, a database row, anything the watcher isn't looking at.

## The Sharp Corollary: Cadence Changes Severity

The underlying defect — unbounded retry of a deterministic failure — wasn't new. It had existed for weeks on a timer-driven host where the same job ran once an hour, cost one wasted attempt each time, and nobody noticed. Converting the job to event-driven, watching the folder instead of polling it, turned one attempt an hour into one attempt every ten seconds. Nothing about the retry logic changed. Only the cadence did, and the cadence is what turned a shrug into an incident.

**Going event-driven is not a free latency win. It's a dependency on having bounded your failure modes first**, and the two pieces of work here were planned in the opposite order — the trigger got faster before the retry got safer. A job that's sloppy about repeating itself can hide behind a slow cadence for a long time. Speed the cadence up and the sloppiness is the first thing that shows.

## The Test That Finds It

Not a backoff timer, a file count. Take a count of the watched directory before and after a skipped scan and assert it's unchanged. A lock file, a temp file, and a refreshed marker each pass a "did it write the wrong output" check while still re-arming the trigger — the count is the only thing that catches all three, because it doesn't care what the write is for, only that it landed inside the surface being watched.

## Related

- [[one-authority-for-repeating-behaviors]] — same family, unbounded repeat with a governor as the fix; there it's several uncoordinated retry loops, here it's a single trigger re-arming on its own exhaust
- [[monitor-cannot-see-its-own-exhaust]] — the read-side twin: a monitor misreads its own artifacts as someone else's signal, here a trigger re-arms on its own artifacts instead
- [[an-event-is-not-a-cause]] — the same event-driven trap from the other direction: an event fired for the wrong reason gets acted on as though it meant what it usually means

## Source

The personal agent's local transcription pipeline: a launchd `WatchPaths` agent scanning a voice-capture folder synced across two machines. A silent, no-speech recording failed deterministically on every scan; the scanner's own failure marker, written beside the audio, re-armed the watcher on every write. 145 attempts in 16 minutes, 144 failures, one speech-model invocation per attempt, throttled only by launchd's ~10s floor, and 21 sync-conflict copies of the marker manufactured across the two hosts. Found while dogfooding the two-machine setup, 2026-08-05.
