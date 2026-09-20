---
type: pattern
date: "2026-07-20"
source: The personal agent session (2026-07-20). A fleet agent (the household agent) was sending repeated "out of credits" status alerts that reached an end user (a family member) over Telegram; investigating the fix surfaced the general discipline for proactive agent notifications.
tags:
  - notifications
  - alerting
  - agents
  - orchestration
  - ux
---

# Quiet-by-Default Notifications; Route Alerts to the Operator

An agent that can message people will, left alone, message them too much. Proactive notifications — reminders, status alerts, "something happened" pings — are useful once and noise thereafter. The discipline is three rules: fire on *change* not on a *timer*, send operator-actionable alerts to the *operator* not to *end users*, and fix the *trigger* not the *message* when something spams. Quiet by default; loud only when a human needs to act.

## The Problem

Proactive messaging degrades in predictable ways:

- **Timer-driven repetition.** An alert wired to a schedule ("check every 5 minutes, notify if bad") re-sends the same bad news every cycle until the condition clears. The first message informs; the tenth trains the recipient to ignore the channel.
- **Broadcasting operator concerns to end users.** Infrastructure status — "out of credits," "provider down," "quota low" — is actionable by the *operator* and meaningless to an *end user*, who can't fix it and didn't ask. Forwarding it to everyone on the allow-list turns an ops signal into user-facing spam.
- **Muting the symptom instead of the cause.** When an alert spams, the reflex is to suppress the message. But an alert firing repeatedly usually means its *underlying condition* is still true. The real fix removes the trigger (refill the credits, fix the provider), which silences the alert as a side effect and fixes the actual problem.

## The Pattern

**Make proactive notifications quiet by default, and separate operator alerts from user messages.**

- **Fire on state change, with a cooldown.** Send once when a condition flips (OK→bad), once when it recovers (bad→OK), and enforce a minimum interval before the *same* alert can repeat. A lockfile or a `last_sent` timestamp is enough. Never re-emit on every poll.
- **Route by who can act.** Operator-actionable alerts (billing, health, quota) go to the operator only. End users get messages that are *about them and actionable by them*, never infra status. Default the notify-target to the operator; broadcasting to all users is an explicit, rare choice.
- **Fix the trigger, not the message.** Before adding suppression, ask what condition is firing the alert and whether *that* is the thing to fix. Suppression is for genuinely noisy-but-correct signals; a repeating alert about a real, fixable condition wants the condition fixed.
- **Tether notification policy to the same gates as everything else.** In a tiered/gated system, the outbound-message path is a send surface — it rides the same deterministic send-gate as any other egress, so notification discipline can't become a second, ungated channel.

## Why It Works

- **Change-plus-cooldown carries all the information at a fraction of the volume** — you learn when something breaks and when it recovers, without the noise in between
- **Routing by actionability keeps every channel meaningful** — the operator's channel stays signal because it isn't diluted with user chatter, and the user's channel stays trusted because it never carries ops noise they can't act on
- **Fixing the trigger solves the real problem** — the spam and the underlying fault die together, instead of hiding a live fault behind a mute

## When to Use

- Any agent or job that sends proactive messages: reminders, health/status alerts, cost warnings, digests, nudges
- A fleet where multiple agents can each message the same humans
- Systems with both an operator and end users on the same messaging transport (e.g. one Telegram bot serving both)

## When NOT to Use

- Strictly pull-based interfaces where the human asks and the agent answers — there's no proactive channel to discipline
- Genuinely per-event notifications the user explicitly opted into and wants each of (a chat reply, a requested confirmation). Don't dedup away messages that are the product

## Watch-outs

- **A router coordinating a `delegate-to-agent` worker's notifications can't reach into its runtime.** The cadence knob may live in the worker's own repo (editable via a reviewed change) *or* only in its deployed runtime config (the operator's to flip). Locate which before promising a fix — some of it may be structurally out of the router's hands
- **"Fire on change" needs a stable definition of the state.** A flapping condition (credits hovering at the threshold) can still spam if each flap counts as a change — debounce the state, not just the message
- **Recovery messages are notifications too.** "All clear" is welcome once; on a flapping signal it's as noisy as the alert. The cooldown must cover both directions
- **Refilling/fixing the trigger is the *fastest* de-spam, and it's easy to skip** because it lives in a different system than the alert code. Check it first

## Adjacent Patterns

- `thin-router-orchestrator.md` — why a router coordinates a delegate-to-agent's behavior through its source/bounded tasks, never its runtime
- `router-worker-exfil-containment.md` — the send-gate this notification path must ride, so alerts aren't a second ungated egress
- `registry-based-monitoring.md` — the monitor side that *generates* status signals a notifier must then throttle
- `morning-briefing-pipeline.md` — the batched, pull-shaped counterpart: aggregate into one scheduled digest instead of many event pings

## Source

The personal agent session, 2026-07-20. The household agent (a fleet agent on a Raspberry Pi, `delegate-to-agent` from the router's view) was sending repeated "out of credits" status alerts that reached an end user, a family member, over Telegram. Investigation found the alert engine itself was already disciplined — deduped via a lockfile, firing on outage state-change rather than on the 5-minute poll. The genuine spam came from two places the message wasn't the fix for: the alert was being *broadcast to all allow-listed users* (a routing rule living in the agent's deployed runtime, not its repo — so the operator's to change, not the router's), and the underlying condition was a real, fixable one. The durable fix for the specific complaint wasn't muting — it was refilling the credits, which removed the trigger entirely. The general shape: quiet by default, route operator alerts to the operator, fix the trigger before suppressing the message, and keep the notification path on the same gates as any other send.
