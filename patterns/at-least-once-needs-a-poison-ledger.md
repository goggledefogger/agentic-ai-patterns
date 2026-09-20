---
type: pattern
date: "2026-08-18"
source: The personal agent's agent board — the household property lane (board_intake Story 1.4, voice-autotriage's documented unbounded retry, and the redo lesson from the same build day)
tags:
  - code
  - reliability
  - messaging
  - agents
---

# At-Least-Once Needs a Poison Ledger

A consumer of an agent-to-agent task queue re-attempts what did not finish — that is what makes it at-least-once instead of best-effort. But re-attempting is only safe with three pieces of state, and each has a documented failure when it is missing. Owed to this library since 2026-08-09; paid by the build that finally needed all three at once.

## The three pieces

- **An idempotency key on the WORK, not the message.** The same job arrives twice — two channels, a re-send, a duplicate alert. Key on what the work produces (the listing's address slug, not the message id), so the second arrival finds the first result and answers "already done" instead of doing it again. Belt-and-braces is legitimate: the producer dedupes on its envelope key AND the consumer dedupes on its derived slug, because either side alone can be bypassed by the door the other one watches.

- **A failure ledger that counts UP and is kept.** The tempting shape is to drop the failure record so a transient breakage cannot strand a task forever. That is exactly voice-autotriage's documented bug: a discarded `failed` key meant a *permanently* failing note re-attempted **96×/day with no cap, backoff, or counter, until a human noticed** — the worst case was unbounded-until-noticed, not a ceiling. Count attempts, persist the count across restarts, and at a small cap (3) mark the task poisoned — labelled for a human, never re-attempted. A poisoned task is a *finding*; a silently retried one is a bill.

- **A deliberate override for redo, separate from retry.** Idempotency that stops duplicate work also stops a better redo when the *method* improves — a thin early result would be frozen forever. The override must be an explicit verb on the task ("re-evaluate"), never a loosening of the seen-check, and any diagnostic (`--dry-run`) must apply the same override logic as the real path, or it reports the opposite of what the next run will do. Measured: it did.

## The boundary conditions that were measured, not theorized

- **A consumer proved against an empty inbox is not proved.** The reader was validated live and reported "inbox empty — pass." The label it queried had never been created; it would have said the same with a hundred tasks waiting. Prove a consumer with a real task through it, end to end.
- **Refusals are terminal, not failures.** A malformed or out-of-scope task is left for a human with its reason, and is NOT charged against the retry budget — it did not fail; it was never eligible.
- **The failure stop must be honest about partial work.** Stop the sequence at the first failed step so nothing downstream (publish, notify) runs on half-done work; the retry re-enters from the top, and idempotent legs make the re-entry cheap.

## Relation to arrival accounting

`no-delivery-without-arrival-accounting` covers the send side: how the producer learns its output landed. This pattern is the consume side: how the worker avoids doing landed work twice, and how a task that can never succeed stops burning money. A queue needs both; each was built in the same lane a week apart because having one made the other's absence visible.
