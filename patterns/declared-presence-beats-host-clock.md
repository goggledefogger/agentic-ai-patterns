---
type: pattern
date: "2026-07-31"
source: The personal agent timezone handling — the operator's laptop stays home on Pacific while he travels, so the host clock silently mis-times every relative reference in a brief
tags:
  - agent-behavior
  - timezone
  - dates
  - travel
  - verification
---

# Declared Presence Beats the Host Clock

An agent must never infer which timezone its user is in. Presence is declared, with an expiry, or it is unknown, never derived from wherever a machine happens to sit.

The obvious design assumes the laptop travels with the user, so the host clock is the answer. The author's actual setup breaks that assumption: his Mac stays home on Pacific while he is physically on the east coast. The host clock is then a fact about an empty room, and every relative time in every brief is silently wrong by three hours. Nothing in the output looks wrong, "your meeting is at 2pm" reads exactly like a correct sentence.

## The Pattern

1. **A committed declaration file**, not per-host, not gitignored, holding an IANA zone and an inclusive `until` day. Committed because a laptop, a server, and a cloud session must all answer "where is the user" identically, a gitignored or per-machine file lets two of the three disagree.
2. **Precedence, strictly in this order**: a hard env override, then a live declaration, then a configured home zone, then the host clock last. The host is usually a datacenter or a machine that stayed behind, so it answers last, not first.
3. **The expiry is the reliability feature, not a chore.** A trip that ends stops steering the clock on its own, a forgotten declaration self-heals back to home instead of silently mis-stating times for months.
4. **Ask, never auto-switch.** Google Calendar ships exactly this shape: detect a likely new timezone, prompt, and keep home if the user declines. Detection earns a question, never a silent switch.
5. **A lapsed declaration is worth exactly one question.** The trip is over by the calendar but the file still claims otherwise, the agent is about to change which zone it speaks in without being asked, which is the one moment a lapsed value must interrupt instead of quietly expiring.

## Why It Works

IP geolocation, connection origin, and guessing from calendar event timezones are the same mistake with better production values, each infers a body's location from a machine's, and a machine is not a body. Published industry guidance already warns that IP detection breaks on VPNs and travel; a smarter inference is still an inference, and every inference fails silently in exactly the case that matters, the traveling user. A declaration doesn't infer. It states a fact the user provided, with a deadline that limits how wrong it can go before it heals itself.

## When to Use

- Any agent computing relative times ("today," "in an hour," "your 2pm") for a user whose device location and physical location can diverge, laptops left at home, shared servers, cloud sessions.
- Deciding where a timezone should live: a committed file the user edits deliberately beats a header, a cookie, or an IP lookup every time.
- Reviewing a "smart" timezone-detection feature, check whether it asks before switching, and whether a declaration ever expires back to a default.

## Adjacent Patterns

- `current-date-grounding.md` — same family: an agent stating a wrong time with total confidence. That pattern re-anchors *which day* it is; this one re-anchors *whose clock* it's reading from.
- `replica-freshness-travels-with-the-count.md` — same shape, an uncertain reading must announce its uncertainty instead of rendering as a clean number. A lapsed timezone declaration is exactly that: a value that was true and is now unverified, not a value to keep trusting.

## Source

The personal agent — `scripts/roytime.py` and `registry/whereabouts.json`, the agent's own repo, 2026-07-31.
