---
type: pattern
date: "2026-10-23"
source: An unattended property-evaluation pipeline, the day a listing site rate-limited the host and the photo step failed the same way on every poll
tags:
  - code
  - reliability
  - agents
  - security
  - playbook
---

# A Roadblock Ledger the Runner Writes, the Worker Reads

An unattended pipeline had two kinds of parts. A research worker (an LLM) read and rewrote per-region playbooks: dead endpoints, working recipes, what to try first. Fixed code legs expanded a share link, fetched listing photos, and published. The playbooks learned. The code legs did not. When the listing site rate-limited the host, the photo leg failed with "no gallery images found", the 10-minute poll retried, and the board issue got the identical "picked this up" claim comment twice, with no reason in either. It would have spent all 3 retries in 30 minutes against the same wall and then parked with a label and nothing else. A new roadblock simply failed the same way until the budget ran out.

The cause is that learning went to whoever was a model. Deterministic legs hit most of the walls, and they had no channel to say so.

## The Pattern

1. **The runner owns a roadblock ledger; the worker only reads it.** On any failed step the runner upserts one row per (step, reason): first seen, last seen, count, and the last date the whole pipeline fully succeeded. That last field turns "worked before, not now" into a date. Rows expire after 14 days without recurrence, and the table is capped at 20 rows, so it cannot grow into the next unread file.

2. **Reasons are fixed codes, never raw error or page text.** The worker is told to follow this file, so anything free-form in it is a persistent prompt-injection path: a hostile page produces an error string, the runner copies it into the ledger, and every later run reads it as instruction. A security review of the first version caught exactly this. Codes like `rate_limited` or `no_gallery` carry the information and none of the attack surface.

3. **The worker routes around what is listed.** Told the photo step is walled, it found the gallery on the brokerage's own page and emitted `PHOTOS-URL:` lines. The fixed leg tries those only as a fallback to its own path.

4. **A URL that came from fetched content is untrusted input to the fetcher.** The fallback made the worker's output a fetch target, which is a server-side request forgery vector. Every fetch, and each redirect hop, must resolve to public addresses only. The same review caught this too.

5. **A rate limit is a state, not a failure.** Back off for a window (2 hours here) without spending the retry budget, since the retry cannot succeed until the window passes and counting it only parks work that was fine.

6. **A retry that cannot say why is noise.** Every retry comment names the last failure in one plain line, and a parked job gets a comment with the reason, not only a label. Without that, a human cannot tell a wall from a bug.

## When It Shows Up

Any pipeline where a model learns through a playbook and deterministic steps sit beside it: scrapers, publishers, sync jobs, enrichment stages. The tell is a playbook full of lessons about the world next to code that fails identically on the third attempt. The writer split is the point: code owns the ledger (codes, counts, dates, expiry) because only code sees every failure and only code can be trusted not to copy hostile text, and the model reads it because routing around a wall is judgment.

This is the code-leg counterpart of [`appending-is-not-learning.md`](appending-is-not-learning.md), which keeps a learning channel from growing without bound; the expiry and row cap here are the same budget idea applied to a ledger. It extends [`at-least-once-needs-a-poison-ledger.md`](at-least-once-needs-a-poison-ledger.md): that ledger counts attempts so a task stops retrying, while this one records why, so the next attempt can go a different way and a rate limit does not count as a strike.
