---
type: pattern
date: "2026-06-29"
source: The personal agent chief-of-staff router session — a 28→29 midnight rollover left "tomorrow" stale for today's meetings
tags:
  - agent-behavior
  - dates
  - accuracy
  - briefing
---

# Current-Date Grounding

An agent must anchor every relative time reference — "today", "tomorrow", "this morning", "yesterday" — on the real current date the runtime reports *at the moment it speaks*, not the date the session opened on. Relative dates are a latent bug: correct when written, silently wrong an hour later.

## The Problem

A long session crosses midnight. The agent built its model on Sunday — "your meeting is *tomorrow*" — and kept repeating "tomorrow" into Monday, when those meetings had become *today*. Nothing on screen looks wrong; the word "tomorrow" reads fine. But it now points at the wrong day, and for a chief-of-staff agent that means telling the user a real meeting is a day off — the error class that makes someone miss or mistime it.

Concrete instance: The personal agent (a router / chief-of-staff) was asked on the 28th to "prep tomorrow's Business Time meeting." The session ran past midnight into the 29th. The personal agent kept framing the Business Time meeting and a downstream interview as "tomorrow" — both were now *today*. The runtime had even injected a "the date has changed" notice, but the relative references already in flight weren't re-anchored until the user caught it.

Two compounding traps:

- **Relative dates in saved artifacts.** A brief, note, or filename that says "tomorrow" is read later, when "tomorrow" means a different day. The word doesn't travel with its meaning; an absolute date does.
- **Timezone-stripped times.** A meeting at "8:00" is ambiguous across ET/PT. A date without its timezone can land on the wrong day near a midnight boundary.

## The Pattern

1. **Read the runtime's current date at the point of any temporal claim**, not once at session start. Treat "what day is it" as a live lookup, not a cached fact.
2. **Re-anchor on a date-change signal.** When the runtime reports the date rolled, immediately re-evaluate every pending "today / tomorrow / this week" reference — in drafts, in briefs, and in your own next sentence.
3. **Prefer absolute dates in anything durable.** "Monday, June 29" over "tomorrow" in briefs, filenames, messages, commits. Relative words are fine in live throwaway chat, never in an artifact read later.
4. **Resolve relative → absolute against the live date, and verify a scheduled item against its own timestamp and timezone** before stating its day.

The discipline is cheap and the failure is expensive: a wrong day isn't a small slip, it's the user standing up for a meeting at the wrong time. Companion to `self-reporting-staleness-check.md` (which catches *past-due* dates in a corpus) — this one guards *relative* dates in live output.
