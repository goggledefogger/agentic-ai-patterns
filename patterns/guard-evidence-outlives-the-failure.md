---
type: pattern
date: "2026-08-02"
source: The agent's own repo, 2026-08-02. The one cron that detects streaming-provider outages wrote its log to /tmp. Every reboot wiped it, so a 19-run failure streak read as starting on the last boot date — the log's own start line was mistaken for the outage's start, in a catchup report, before anyone noticed the two dates were the same for a reason.
tags:
  - monitoring
  - detectors
  - verification
  - logging
  - anti-pattern
---

# A Guard's Evidence Must Outlive the Failure It Detects

A smoke test ran every six hours against streaming providers and wrote its output to `/tmp/media-downloads/smoke-test-cron.log`. It worked. It correctly detected that one provider had gone dry for TV content, and correctly emitted `FAILURES DETECTED` on every run.

Then someone asked the only question that matters about an ongoing failure: **how long has this been happening?**

The log's first line was dated four days earlier. That is not when the outage began — it is when the machine last rebooted and `/tmp` was cleared. The evidence had been silently truncated to "since the last reboot," and the truncation left no mark: no gap, no rotation header, no note. Just a log that begins confidently at a date that means nothing.

## The Problem

A detector answers *is it broken right now?* Its log answers a different and often more important question: *for how long, and starting when?* The second question is what decides whether something is a blip, a regression with a known trigger, or a slow degradation nobody noticed.

The trap is that the two questions have different durability requirements, and only the first one is ever tested. A guard writing to volatile storage passes every test you would think to write:

- It detects the failure. ✅
- It logs the failure. ✅
- A human reading the log sees the failure. ✅
- The log survives the failure it is detecting. ❌ *never checked*

Worse, the volatile log **fabricates a plausible answer** rather than admitting ignorance. A log starting on the reboot date looks exactly like an outage starting on the reboot date. That is a specific, dated, confident claim — and it is an artifact of the storage medium. Reboots correlate with real incidents (crashes, power events, upgrades), so the fabricated start date will often land suspiciously close to a real event and *confirm* a wrong causal story.

There is a second-order version: the failure being detected can itself be the thing that destroys the evidence. A guard that watches for crashes, and logs to a location a crash clears, is at its blindest exactly when it matters.

## The Fix

**Store a guard's output at least as durably as the longest failure it is meant to characterize.** If the detector is supposed to catch a slow degradation over weeks, a log that resets on reboot cannot do the job, no matter how correct the detection logic is.

Three checks, cheap to apply:

1. **Where does this land, and what clears it?** `/tmp`, a container's writable layer, a RAM disk, a directory some cleanup cron prunes by age. Any of these means "since the last time something unrelated happened."
2. **Does the artifact carry its own coverage window?** A log whose first line could mean *"this is when the problem started"* or *"this is when the file started"* should say which. A header line naming the retention or rotation policy converts a fabricated answer into an honest one.
3. **Is there already a durable copy?** Often yes, and the volatile one is redundant. In the case above, the script itself already wrote a durable artifact to a persistent workspace path going back five months — only the cron's stdout redirect was volatile. The fix was a path change in one crontab line, not new infrastructure.

## Related

- `unrun-checks-read-as-passing.md` — a guard that didn't run looks like a guard that found nothing. This is the sibling: a guard whose *evidence* didn't survive looks like a guard reporting a shorter problem.
- `replica-freshness-travels-with-the-count.md` — an answer computed from a stale source carries a hidden "as of" qualifier that its shape doesn't show. A truncated log carries a hidden "since" qualifier the same way.
- `self-report-must-not-end-what-it-reports-on.md` — the general shape of a reporter whose own mechanics corrupt what it reports.
