---
type: pattern
date: "2026-07-29"
source: The work vault — 194 coaching notes carrying ~456 never-reviewed vault suggestions, measured 2026-07-29
tags:
  - feedback
  - discipline
  - meta
  - vault-hygiene
---

# Embedded Channels Inherit the Host Document's Read Rate

When you nest one artifact inside another — a suggestion list at the bottom of a review, an action-item block inside a retro, a TODO section in a status report — the nested artifact inherits the host's read rate. If the host is read late, skimmed, or avoided, the nested content is too, no matter how actionable it is. The fix is to give the nested artifact its own file and its own review cadence, and leave a pointer behind.

## The Problem

The failure is invisible because both artifacts are individually well-formed. The host document gets written. The nested section gets written. Nothing errors. The generator keeps generating, on the reasonable assumption that producing the content is the job.

Concrete instance. A meeting-transcript skill produced a coaching note after every meeting: honest feedback on how the user showed up, ending with a `## Vault suggestions` section — durable patterns worth promoting into skill files and conventions. The coaching layer worked exactly as designed. Over roughly four months it produced **194 notes carrying ~456 suggestion bullets**.

The measurement, when finally taken:

- **194 of 196 notes still on disk.** The documented lifecycle was read-then-delete; nothing had been deleted, so nothing had been processed
- **~456 suggestion bullets, zero systematically reviewed**
- **87 bullets targeted a file that had been reduced to a stub months earlier.** The largest single suggestion stream pointed at a dead path and nobody noticed — the signature of a channel with no reader

The user's own diagnosis named the mechanism: *"sometimes I'm hesitant to read the coaching output."* Coaching notes are critical feedback about you. Process suggestions are cheap, impersonal, and useful. Coupling them meant the cheap useful thing inherited the emotional cost of the expensive uncomfortable thing — and inherited its avoidance.

Emotional weight is the sharpest version, but it is not the only one. Any read-rate mismatch does this: a weekly-cadence action list nested in a quarterly report, an urgent flag inside a document read for reference, a decision log at the bottom of a meeting transcript nobody reopens.

## The Pattern

**Match each artifact's home to its own review cadence, not to the cadence of whatever produced it.**

Three steps.

### 1. Notice the mismatch

The tell is a generator that reliably produces and a consumer that never consumes. Ask of any nested section: *would this be read if the host document did not exist?* If yes, and the host is read less often than the nested content deserves, they should be separate files.

The stronger tell is a **verifiable staleness signal in the nested content** — suggestions pointing at renamed files, action items referencing closed tickets, dates long past. Content decays like that only when nobody is reading it.

### 2. Split the channel, leave a pointer

Move the nested content to its own file with its own rules. The host keeps a one-line pointer: `Vault suggestions routed to [[vault-suggestions]] (2 new, 1 evidence-bumped).` That preserves discoverability from the host and costs one line.

The destination file wants three things:

- **A queue shape**, typically Open / Done / Declined, so review is triage rather than reading
- **Dedupe by evidence-bump.** When the same suggestion recurs, increment a count on the existing row instead of appending a duplicate. This flips repetition from bloat into signal: a 5× row has earned attention, a 1× row can wait
- **Declined as memory.** A rejected item moves to Declined rather than being deleted, so the generator stops re-proposing it

### 3. Keep it a proposal channel, not a write path

The tempting next step is to have the generator apply suggestions directly. Don't. The reason the backlog was safe to ignore is that nothing had been auto-applied — 456 unreviewed bullets sat inert instead of silently reshaping the corpus. Auto-application converts an unread queue into unreviewed drift, which is strictly worse. **The queue proposes; a human adopts.** Pair with `appending-is-not-learning.md` for the budget discipline once items start landing.

## Why This Beats "Read Your Notes"

The instinct is to fix the human: read the coaching output, process the queue. That fails for the same reason it failed for four months — the avoidance is structural, not a discipline gap. Someone hesitant to read feedback about themselves will stay hesitant. Splitting the channel removes the coupling instead of asking the person to overcome it, and the process improvements start flowing at their own cadence.

## Verification

The check is a **count of the source, not the survivors** (see `count-the-source-not-the-survivors.md`). Do not ask "are suggestions landing?" — ask "how many were produced, and how many were acted on?" The gap is the answer. Here: 456 produced, 0 acted on, which no amount of qualitative confidence would have surfaced.

Cheap re-checks, both of which the split makes possible:

- **Lifecycle violation as a proxy.** If artifacts with a documented read-then-delete lifecycle are accumulating instead, the read is not happening. File count is the metric, no content analysis needed
- **Dead-target scan.** Grep the queue's target paths against the filesystem. Rows pointing at files that no longer exist prove the queue has gone unread since those files moved

## Related

- `wire-into-existing-flows.md` — the split creates a new artifact, which needs its own forcing function or it becomes the next unread file. Wire the queue into an existing ritual in the same commit
- `appending-is-not-learning.md` — governs what happens once items are adopted, so the destination files don't accrete
- `count-the-source-not-the-survivors.md` — the measurement discipline that surfaces this class of failure
- `living-doc-archive-split.md` — the same split move for size rather than read-rate mismatch
