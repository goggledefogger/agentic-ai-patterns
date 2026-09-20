---
type: pattern
date: "2026-08-17"
source: The personal agent's credential checker (auth_health.py) — a personal Gmail OAuth lane that is dead by construction until an unrelated piece of work happens, sitting in the same table as nine credentials whose death would matter today
tags:
  - monitoring
  - alerting
  - anti-pattern
  - observability
---

# A Known-Broken Lane Must Stay Visible and Must Never Page

Some things are broken on purpose, or broken pending a decision nobody has made yet. They belong in the status table, because a gap you cannot see is a gap you never close. They do not belong in the alarm, because an alert that fires every day for a reason nobody intends to act on trains the reader to ignore the channel — and the channel is shared with the failures that do matter.

## The Problem

A credential checker covered ten lanes. One of them — a personal Google OAuth token — was dead for a structural reason: the OAuth client it uses belongs to a Workspace and cannot authorize an outside account. Fixing it means creating a separate cloud project, a decision that had been deliberately deferred as *not currently wanted*.

The naive checker reports it DEAD every morning, forever, on the same push channel that carries "your bot token was revoked" and "the backup credential is gone."

Both available naive options are wrong:

- **Page on it.** Every morning the digest contains a red line nobody will act on. Within a week the reader skims past the whole section. The next *real* failure arrives into a channel that has been trained to be ignored, and the alarm's own history is the thing that silenced it.
- **Delete the row.** The gap becomes invisible. Six months later somebody asks why personal mail cannot be read from a scheduled job and rediscovers the whole reasoning from zero — and worse, nothing records that this was a *choice* rather than an oversight.

The second failure is the sneakier one, because deleting the row feels like tidying.

## The Pattern

**Split "is it healthy" from "should someone be woken."** They are different questions and a single boolean cannot answer both.

1. **Keep the row in the table, with its true state.** DEAD stays DEAD. Do not soften the verdict to keep the table green — that is lying to the reader to protect the alarm.
2. **Mark it expected, with the condition that ends the exemption.** Not "ignore this" but "expected dead **until a personal OAuth client exists**". The exemption names its own expiry, so it cannot quietly become permanent.
3. **Exclude it from the alarm predicate, not from the report.** In code: `failing()` filters the expected ones out; `render()` still prints them, under their own heading, with the reason.
4. **Print the exempt count in every summary, including a clean one.** A summary that mentions known-broken lanes only when something *else* failed hides them on exactly the days someone is reading a healthy report.
5. **State the reason for every non-check, not just the exemptions.** "No liveness check" with no cause reads as an oversight, and the fix someone reaches for is a hastily written probe. "Mac-only — this store is not mounted on this host" reads as a boundary, and gets left alone.

## Why It Works

The alarm's value is entirely its signal-to-noise ratio, and that ratio is not a property of any single alert — it is a property of the channel's history. One daily false-positive is enough to retrain a human, and retraining is not reversible by fixing the alert later; the reading habit persists.

Meanwhile the table's value is completeness. It is read deliberately, by someone who came looking, and its job is to answer "what is the true state of everything." Those two jobs pull in opposite directions on exactly one class of row, and the resolution is to serve both rather than pick.

Naming the expiry condition is what stops this from becoming a suppression list. A suppression list accumulates and is never revisited; an exemption that says *until X exists* is a to-do with a trigger, readable by anyone.

## Trade-offs and Limits

- **The exemption must be per-row and justified in the row**, never a global mute flag. A mute flag is a suppression list wearing a better name.
- **Re-examine exemptions when the condition changes.** If the personal OAuth client gets built and the row still says "expected dead", the checker is now lying in the other direction.
- **A row exempt for a long time is a signal in itself.** If nobody has acted on the condition in months, either the lane is not wanted at all and the row should be deleted with a note, or the deferral needs re-litigating. Silence is not consent from a monitoring table.
- **This is not the same as flapping suppression.** Flapping is a noisy *real* signal and wants hysteresis. This is a *stable, understood, intentional* failure and wants an exemption with a named end condition.
