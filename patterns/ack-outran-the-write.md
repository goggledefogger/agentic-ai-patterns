---
type: pattern
date: "2026-08-08"
source: The personal agent cloud-VM leader — a Telegram voice note was consumed, its download failed, and the poller acked the update anyway; Telegram deletes a confirmed update, so the recording is unrecoverable, 2026-08-08
tags:
  - correctness
  - data-loss
  - anti-pattern
  - reliability
  - observability
---

# The Ack Outran the Write

A poller drained a message queue. For each update it advanced its cursor, then wrote the payload to disk. Those two steps were one line apart and in the wrong order, and the gap between them was where a voice recording died.

The cursor advanced at the **top** of the loop. The download happened eight lines later. When the download failed, the code appended a line to a `skipped` list and `continue`d — with the cursor already moved. The next poll sent the new cursor, the upstream took that as confirmation, and deleted the update. There is no copy anywhere. It is simply gone.

```python
for u in updates:
    offset = max(offset or 0, u.get("update_id", 0))   # ← ack, unconditional
    ...
    data = downloader(fid)
    if data is None:
        skipped.append(f"{dest.name} (download failed)")
        continue                                        # ← no write, still acked
```

## The Pattern

**An ack is a delete on someone else's server.** A cursor, offset, checkpoint, or "confirmed" flag is not bookkeeping — it is an instruction to a system you do not control to forget something forever. It must therefore be the *last* thing that happens, strictly after the durable write it certifies, and it must never advance past work that did not succeed.

The rule is simple to state and easy to violate, because the ack is usually the cheapest line in the loop and reads like loop maintenance rather than an irreversible external effect.

Three properties make this class of bug uniquely nasty:

- **It is unrecoverable.** Every other bug in the loop can be fixed and re-run. This one destroys its own input. Retry logic, backfills, and replays all depend on the upstream still having the data.
- **It only fires when something else is already broken.** The download failure was the trigger; the ordering was the defect. On a healthy network the code is correct-looking and correct-behaving for months.
- **Fixing the trigger hides it.** The same session pinned the transport to IPv4, which took the download failure rate from ~25% to zero. Had that been the only change, the loss would have stopped happening and the hole would still be there, waiting for the next `getFile` timeout. **When you fix a trigger, go looking for what the trigger was exposing.**

### The single-watermark trap

A cursor is one integer, so acking update N+1 implicitly acks N. That makes "skip the failure and keep going" impossible to express: you cannot ack a later item while holding an earlier one back. The only safe watermark is **one past the last item before the first failure** — stop dead, do not step over.

Everything after the stuck item gets re-delivered next cycle, which is only tolerable because the writes are idempotent (an existing file is never rewritten). **The hold-back rule and the idempotent write are one design, not two.** Adding the first without the second converts silent data loss into silent duplication.

### Resolved is not written

Getting this wrong in the other direction produces a liveness bug that looks just as bad. Plenty of items will *never* be written and must still be acked: a message from a stranger, an update type you don't handle, a payload with nothing capturable in it. If "not written" blocks the cursor, the poller wedges forever on the first item it was never going to store, and everything behind it starves.

The predicate is not "did we write it" but **"could a later attempt do better?"** Only a recoverable failure blocks. Name the two categories explicitly in code, because the default reading of a `skipped` list merges them.

## The Three Silences

The loss printed as `0 captured` — character-identical to a poll that found nothing. Three independent silences stacked up, and any one of them alone would have made this visible:

1. **The swallow.** `except Exception: return None` turned a TLS reset into the same value as "no file here." The caller could not distinguish a network fault from an empty result, so it could not decide whether to retry.
2. **The discard.** The caller took `written, _, _ = capture(...)` — throwing away both the `skipped` list *and* the cursor the callee had carefully held back. A function computing the right answer is worth nothing if the caller drops it.
3. **The tie.** The success metric ("captured: 0") had the same value in the healthy case and the catastrophic one. A counter that cannot distinguish "nothing happened" from "something was destroyed" is not instrumentation.

### The confirmation lag reads as health

One more trap, and it is the reason an observer watching from outside disbelieved the loss for ten minutes. The upstream's delete is **lazy**: the update is not actually confirmed when the client stores its new cursor, but when the *next* request carries that cursor. So for the eleven minutes between the failed poll and the following one, the queue still reported the item as pending. Someone checking "is it still there?" got **yes** — while the item was already condemned by a cursor sitting in a local file.

**The window between "we decided to forget it" and "they forgot it" reads as healthy and is the exact window in which the decision is still reversible.** If you catch it there, you can fix the cursor and save the data. Once the next poll goes out, you cannot. Check the client's stored cursor, not the server's pending count.

## Where Else This Lives

- Kafka: `commit()` before the handler finishes, or `enable.auto.commit` with slow processing
- SQS: `DeleteMessage` on receipt rather than after the work
- Webhooks: returning `200` before persisting, so the sender stops retrying
- Cursor pagination into an ETL: checkpoint written per page, not per successfully-loaded page
- IMAP/POP: `\Deleted` or `DELE` before the local mailbox write is flushed
- Any "move the file to processed/ then handle it" pipeline

The tell in review: a loop where the position marker is updated anywhere other than the last statement, or a caller that discards a callee's returned position in favour of one it computed itself.

## How to Verify

The test must fail against the old code, or it is decoration. Both layers here were patched back to their original semantics in a scratch copy and the suite re-run:

- cursor advances unconditionally → the "lone failed download acks nothing" assertion fires (`off` was `21`, expected `None`)
- caller ignores the callee's held-back cursor → the veto assertion fires (`53`, expected `51` — and `53` is precisely the value that destroys the recording)

The second one is the one that mattered: the first patch attempt left the caller-side test **passing**, which would have shipped a guarantee that was only half enforced. If you fix a bug at two layers, prove the test bites at *each* layer independently.

## When NOT to Use

- If the upstream retains data independently of your ack (an append-only log with a retention window, an object store you re-list), cursor ordering is a correctness nicety rather than a data-loss risk. Know which regime you are in — "the ack deletes it" is the question to answer before designing the loop.
- Strict ordered hold-back costs head-of-line blocking. Where items are genuinely independent and the upstream supports per-item ack (SQS, JetStream), ack individually instead and skip the watermark discipline entirely.

## Adjacent Patterns

- `fallback-commits-before-the-failure.md`, the transport fault that triggered this, found the same session. Together they are the general lesson: the trigger and the defect are different bugs, and fixing the loud one can bury the quiet one.
- `unknown-value-renders-as-absence.md`, the same "failure is indistinguishable from nothing-to-do" shape, one layer up in a renderer.
- `health-check-that-never-exercises.md`, sibling in the "green means nothing" family.
- `guard-evidence-outlives-the-failure.md`, on keeping the evidence of a failure after the failing run ends.

## Source

The personal agent cloud-VM leader, 2026-08-08. `scripts/telegram_capture.py:capture()` advanced the getUpdates offset at the top of its loop, before attempting the voice-note download; a `getFile` reset on a flaky IPv6 path returned `None` through a bare `except Exception`, the note was skipped, and the update was acked and deleted server-side. `telegram_gateway.py:route()` compounded it by computing its own watermark and discarding both capture's offset and its `skipped` list, so the loss logged as `0 captured`. Fixed by holding the offset at the first unresolved update, having the caller honour that veto, logging skips, and logging the download failure's reason (never the URL — it carries the bot token).
