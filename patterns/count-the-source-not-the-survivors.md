---
type: pattern
date: "2026-07-27"
source: The personal agent — voice-capture inbox, three audio files captured but never transcribed, found 2026-07-27
tags:
  - pipelines
  - monitoring
  - data-loss
  - verification
  - anti-pattern
---

# Count the Source, Not the Survivors

A pipeline turns A into B. Every count anyone writes is over B, because B is the thing with structure — records, front matter, IDs, a directory that means something. An A that never became a B is not a small B or a broken B. It is *absent from the population being counted*, and therefore invisible to every number the system reports about itself.

The loss does not show up as a wrong count. It shows up as a **correct count of the wrong population**.

## The Problem

The personal agent's capture path: audio lands in `audio/`, a local Whisper watcher transcribes it, and a transcript appears in `transcripts/` carrying front matter. Everything downstream — pending-triage counts, the dedup guarantee, the routing log, the nudge that tells the author notes are waiting — reads transcripts. That is correct, because a transcript is where identity lives.

On 2026-07-27, three audio files had no transcript: two Telegram voice notes and one phone recording sitting between two neighbours that transcribed fine. They had been there for days. Every count was accurate. "2 notes pending triage" was true. Nothing was broken enough to log an error, because the watcher's design is to *leave audio in place and retry* when transcription fails or returns junk — deliberately, so a transient failure self-heals.

Nothing ever reported a file that kept **losing** that retry. And the two states are indistinguishable from the transcript side:

- A note that was never sent → no transcript.
- A note that was sent, captured, and failed transcription eleven times → no transcript.

They were only found because a human asked an unrelated question and someone thought to list the audio directory by hand.

The retry loop is what makes this dangerous rather than merely incomplete. A hard failure would have raised something. A design that says "leave it, we'll get it next time" converts a permanent loss into an infinitely deferred success, and infinite deferral emits no signal at all.

## The Pattern

**Reconcile the two populations. The loss lives in the difference, and nowhere else.**

```python
def orphans(root):
    known = {n["source"] for n in scan(root / "transcripts")[0]}
    return [p for p in (root / "audio").glob("*")
            if is_audio(p) and not claimed_by(p, known)]
```

Three things this has to get right, each of which is a way the reconciliation quietly stops working:

**1. Match on every identity the pipeline uses, not the one you happen to know.** the personal agent has two writers stamping provenance differently: the Whisper watcher records the path form (`audio/telegram-40.oga`), while the Telegram text capture records the message form (`telegram:40`). Reconciling on one identity reports every file the *other* writer handled as lost. A reconciler that cries wolf is turned off, and then the real loss is invisible again — this time with a decision behind it.

**2. Count it separately from the healthy queue, never folded in.** "2 pending + 3 orphaned" is not "5 pending." A pending note is routable; an orphan cannot be routed at all, because the thing that would make it routable is the step that failed. Merging them implies an action that does not exist.

**3. Exclude known duplicates by the same rule the main population uses.** A sync-conflict copy of an audio file is not a second capture. If the reconciler applies a different dedup rule than the pipeline, it invents losses.

## Where else this shape appears

The tell is a monitored population that sits **downstream of the failure being monitored for**:

- Counting rows in the destination table to verify a migration. A row that failed to transform was never a row there.
- Counting parsed events to check ingestion. A file that failed to parse produced zero events, and zero events is what a quiet hour looks like.
- Counting resolved tickets to measure a queue. A ticket that errored on creation was never in it.
- Counting indexed documents for search health. A document that failed extraction is missing from exactly the index you are asking.

In each, the metric is honest and the question is wrong. Asking "how many made it?" can never answer "did any not?"

## Verification

Do not trust the reconciler because it returns a list — trust it once it has independently found a loss you already know about by other means. The personal agent's was checked against three orphans identified beforehand by hand, and had to name exactly those three: no fewer (it must not miss the one whose neighbours are healthy) and no more (it must not flag the note the sibling writer handled under a different identity). A reconciler verified only against synthetic fixtures tends to encode the same assumption the pipeline made, which is the assumption that lost the file.

Then leave the *cause* open. Detection and diagnosis are different jobs, and shipping detection does not fix anything: the three files are surfaced now and still untranscribed, because whether they failed, returned junk, or were never seen by the watcher is not yet known — and the fix differs per cause. Announcing the loss is the deliverable; resist reporting it as resolved.

## Adjacent Patterns

- **A check that never ran reads as passing** (`unrun-checks-read-as-passing.md`) — the sibling. There, work that never executed is reported as healthy; here, an input that never survived is absent from the report entirely. Both are silence mistaken for health.
- **Migration blinds readers** (`migration-blinds-readers.md`) — the same trap of trusting emptiness, arrived at by a different road.
- **Unattended run discipline** (`unattended-run-discipline.md`) — why a retry loop with no escalation is a design smell: the run is quiet precisely because it intends to try again.
- **Smoke tests with real data** (`smoke-tests-with-real-data.md`) — the three orphans were found in production data and would not have appeared in any fixture.
