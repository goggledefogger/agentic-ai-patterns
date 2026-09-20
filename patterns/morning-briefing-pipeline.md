---
type: pattern
date: "2026-05-02"
source: Reddit u/Jonathan_Rivera, "How I use Obsidian as the long-term memory backbone for my AI assistant" (r/hermesagent, 2026-04-23, 704 pts)
tags:
  - automation
  - cron
  - daily-notes
  - delivery
---

# Morning Briefing Pipeline

Two scheduled jobs, ten minutes apart. The first builds today's brief state by reading vault Living files and external sources (calendar, tasks, weather). The second renders that state for the user's preferred channel (Telegram, email, terminal, push). The brief is a *render* of state the assistant can already see, not a fresh synthesis at delivery time.

> **Companion to `templates/catch-up.md`.** That template is a *manual* session-start command — the user invokes `/catch-up` and the assistant pulls repos, syncs the vault, and updates today's daily note. This pattern is the *automated* push variant — a cron fires without any user invocation and lands a brief in their messaging channel. They cover different shapes of the same problem (the user wants to know what changed) and pair well: catch-up at session start, briefing at a fixed time of day.

## The Problem

Manual morning standups don't scale past about a week. Two failure modes:

1. **The user opens the vault, the assistant, the calendar, the task tracker, and assembles the brief by hand.** Works the first three days. By day five the user skips it, and the routine breaks.

2. **The assistant generates the brief inside an ad-hoc chat turn.** It's plausible-looking, but it's *fresh synthesis every time*. The state the brief describes isn't anywhere durable — the calendar API is the source of truth, the assistant's render is a one-off, and the next render may differ for reasons that have nothing to do with what changed in your day.

The fix: separate the *gathering* of brief state from the *delivery* of the brief, on a schedule, with state cached in between.

## The Pattern

Two scheduled jobs, ten minutes apart, with the brief state cached in between.

### Job A — Gather state (e.g. 6:50 AM)

Runs as a cron job (or launchd, or systemd timer — anything fired without human prompting). Steps:

1. Fetch tasks from your task system (Todoist, Linear, GitHub Issues, etc.) — categorize by project (work / personal / side).
2. Fetch calendar events for the day — filter out noise (sleep tracking, lunch holds, recurring blocks the user doesn't care about).
3. Read the vault Living files that the brief should reflect (e.g. `Home.md` open TODOs, project `CLAUDE.md` flagged items).
4. Cache the gathered state to a known location — a JSON file, a scratch markdown, or fields in today's daily note. Whatever Job B can read 10 minutes later.

Failure handling: if the calendar API is down, cache the partial state with a `Schedule unavailable — see calendar` placeholder. The cached state existing matters more than it being complete. Job B will render whatever's there.

### Job B — Render and deliver (e.g. 7:00 AM)

Reads the cached state from Job A and formats it for the user's delivery channel — Telegram, email, Slack, push notification, terminal banner. The brief is shorter than the underlying state: bullets, not checkboxes; categories, not full task descriptions; an overdue count, not the overdue list itself.

Output shape:

```
Morning brief — YYYY-MM-DD

Today (3):
  • Task A
  • Task B
  • Task C

Schedule (4 events):
  09:00 — Team sync
  10:30 — Vendor call
  ...

Overdue: 2 (see vault)
Waiting: 1 (vendor reply)
```

The user reads the brief in 30 seconds and knows what the day looks like. If they want depth, they open the vault.

### If you want the assistant's voice in Job B

Job B as written is deterministic formatting. Wanting the assistant to *write* the brief instead is a fair ask — a status line reads like a machine, and people stop reading machines. That is compatible with this pattern, but only while the cache stays authoritative:

- **Feed the model the cached state and nothing else.** No tools, no fetching. The moment Job B can gather, it *is* failure mode 2 above — fresh synthesis over inputs nobody recorded, drifting for reasons unrelated to the day.
- **Keep the deterministic render as the fallback.** If the turn errors or times out, send the plain format anyway. A brief that always arrives is the entire point; don't make delivery contingent on a model call succeeding.
- **The cache stays the reproducible artifact, not the prose.** Re-rendering an old date yields the same facts and possibly different wording. That's fine — the facts are what anyone goes back for.
- **This makes Job B a scheduled LLM turn, i.e. a standing charge.** Gate it per `scheduled-llm-spend-gate.md`: explicit operator approval, worst-case cost named, bounded run. A brief is that gate's legitimate timer-fired case, not an exemption from it.

### Where the daily note fits

The daily note is *not* the brief and *not* the cached state. The daily note is where the day's session log accumulates throughout the day — meetings discussed, decisions made, issues hit. Per `claude-code-obsidian-guide.md`, daily notes are session logs, not inboxes.

The brief can append a one-line marker to the daily note ("Brief delivered at 7:00 AM — Today (3), Schedule (4)") so the timeline shows the brief fired, but the brief's content lives in the cached-state artifact, not in the daily note. That keeps the daily note clean as a session log and the brief independently reproducible.

## What this enables

- **The brief is reproducible** — if the user asks "what was the plan for Tuesday?" the assistant reads the cached state for that date and renders the same brief format. No regeneration drift.
- **External-source failure doesn't break the routine** — calendar API outage means a degraded brief, not a missing one.
- **Per-channel render** is cheap — same cached state can render terse SMS, detailed email, and full Telegram from one source.

## Watch-outs

**Don't put logic in the brief that should be in Job A.** Filtering sleep-tracking events, deduplicating recurring tasks, labeling overdue items — that all goes in Job A so the cached state is correct. Job B is purely a render. If you find yourself adding logic to Job B, you're masking a bad cache instead of fixing it.

**The cache has to actually be writable.** Cron jobs run as a user; the cache directory needs write permissions for that user. On systems with vault-immutability guards (e.g. an `~/wiki/` chattr +i layer), the cache lives outside the immutable tree by design.

**Calendar event filters drift.** What counts as "noise" changes over months. Periodically inspect what Job A is filtering and add new noise patterns when they appear (commute holds, focus blocks, etc.). Pair with `living-doc-refresh-ritual.md` for the wider drift discipline.

**Don't add a third job.** "Job C runs at noon to update the brief with afternoon changes" is the start of cascade. Either Job A is right or Job B is wrong; fix one of those instead of adding more crons. The exception is an explicit *evening* digest, which has a different purpose (retrospective, not preview).

**Don't conflate the brief with the daily note.** The brief is a render of state, the daily note is the session-log timeline, the cached state is a Job A artifact. Three different things. Conflating any pair of them re-creates the failure modes the pattern was meant to solve.

## Variations

- **Manual companion**: `templates/catch-up.md` runs the same gathering logic on demand at session start. The two are not redundant — the cron pushes a brief at a fixed time; catch-up pulls when the user starts a session. Many days, both fire.
- **Evening companion**: a 9 PM job reads the day's daily-note session log + completed tasks, summarizes for the user, and prompts for anything to log before sleep. Use sparingly — the brief is a pull (user reads in the morning), the evening companion is a push (assistant prompts at night). The push is more disruptive; default to brief-only first.
- **Weekly retro**: Sunday morning runs an extra job that reads the week's daily notes and summarizes patterns — what got done, what slipped, what came up repeatedly.

## When to Use

- You have a daily routine that benefits from a same-time-every-day brief
- You have at least one external source (calendar / tasks / inbox) that's machine-readable
- You have a delivery channel that supports text (Telegram, Slack, email — not voice-only)
- The cost of a missed routine matters (medication schedules, recovery handoffs, household coordination)

## When NOT to Use

- Highly variable schedules where a fixed time doesn't fit (shift work, frequent timezone shifts)
- Solo, one-machine setups where the manual `/catch-up` companion already covers the need
- Projects without a state-gathering surface — if you have nothing to gather, you have nothing to render

## Adjacent Patterns

- `templates/catch-up.md` — the manual session-start companion to this scheduled pattern
- `three-tier-memory-pipeline.md` — Job A is reading from Tier 2 living files and external sources; the daily note (Tier 3) is the timeline, not the brief itself
- `wire-into-existing-flows.md` — the brief is the wiring; if nobody reads it, the cron is decoration
- `living-doc-refresh-ritual.md` — for keeping the calendar-noise filters and category routes from rotting
- `scheduled-llm-spend-gate.md` — required reading if Job B uses a model to write the brief. A daily job is a standing charge; that pattern is the approval gate, and it treats a brief as the legitimate timer-fired case rather than a violation
