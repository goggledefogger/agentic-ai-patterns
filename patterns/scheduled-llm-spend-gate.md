---
type: pattern
date: "2026-07-21"
source: The household-agent repo (a household agent on a Raspberry Pi). An hourly LLM cron set up to "monitor download progress" polled a permanently-stalled download every 60 min, waking a full model turn to output "[SILENT]" — ~$4.45 overnight for nothing, and it tripped the provider's monthly spend cap and hard-downed an unrelated job. It ran three days unseen because the ops snapshot listed the OS crontab, never the agent's own scheduler.
tags:
  - agents
  - cost
  - scheduling
  - determinism
  - reliability
  - claude-code
---

# Gate Scheduled LLM Spend at Creation; Prefer a Deterministic Trigger

A scheduled job that wakes an LLM turn is a **standing charge** — a subscription the agent signed you up for, billed every interval whether or not there was anything to do. Unlike an interactive turn (visible, bounded, paid once), a recurring LLM cron keeps drawing tokens in the dark. The discipline: no token-spending scheduled job without the operator's explicit approval; prefer a deterministic script that only wakes a model when there's genuinely new work; and make the approval step itself ask whether a *local* model could do the job for free.

## The Problem

An agent that can schedule its own recurring work will, in good faith, create jobs that quietly bleed money:

- **A recurring LLM turn is an invisible standing charge.** Each wake re-ingests the full system prompt + tool schemas + context — often 100K–400K input tokens — *before* it does any useful work. Fire that hourly and you're paying a large bill for the privilege of checking whether there's anything to do. Usually there isn't.
- **Polling a condition that never flips never terminates.** "Wake up, check if X is done; if not, stay quiet and check again" is an infinite token loop the moment X gets stuck. The agent goes *silent* (looks well-behaved) while billing every cycle. Silence is not the same as idle.
- **It can take down unrelated work.** Sustained background spend trips hard limits — a monthly spend cap, an account-level rate limit — that then fail *other* jobs sharing the same key. A cost leak becomes an availability incident.
- **It's invisible to the usual dashboards.** Ops snapshots list the OS crontab (`crontab -l`) — which is zero-LLM and harmless. The dangerous jobs live in the *agent's own scheduler* (Hermes cron, a routine store, a task DB), which nothing was enumerating. The leak above ran three days before anyone saw it.
- **The rule is usually already written — and bends anyway.** "Don't use LLM crons for monitoring" tends to exist in a memory file already; it bent under a warm, concrete request ("help a family member get her episodes"). Prose rules bend; a gate and a backstop don't.

## The Pattern

**Two gates and a default, in this order.**

1. **Deterministic-first (the default).** Before scheduling any LLM turn, ask: could a plain script do this with zero model calls? Most "monitor X and ping me" tasks are deterministic — poll an API, diff a value, `curl` a notification. A 10-line script on the OS cron costs nothing and can't loop into a bill. Only tasks that genuinely need model *judgment* on every run are candidates for a scheduled LLM turn at all.

2. **Creation-time approval gate.** No scheduled job that *can* spend tokens gets created without the operator's explicit approval. Approval is a concrete artifact — an entry in an allowlist file — not a verbal "sure." The approval request must answer, in one short checklist:
   - **Can it be deterministic?** If yes, build that instead; no approval needed for a zero-LLM script.
   - **Could a *local* model do the LLM part?** State it honestly. A local model on the box (Ollama, etc.) turns per-run cost to zero. Caveat it where true — small local models often can't carry a full tool + system-prompt payload, so for tool-heavy turns "deterministic script" usually beats "local model," and for judgment-only text tasks local may be viable. Either way the question is *asked* every time, not skipped.
   - **How often does it wake, and what's the worst-case cost?** Interval × tokens-per-wake × price. Name the number. "Hourly × ~200K tokens" should read as alarming on its face.
   - **What stops it looping?** A hard termination or max-runs, so a stuck upstream condition can't bill forever.

3. **Fire on new work, not on a timer to poll.** Where an LLM turn *is* justified, gate it behind a deterministic check that only wakes the model when there's genuinely something new (the doc changed, the file landed) — never on a bare schedule that wakes to look and usually finds nothing. This is the `deterministic-orchestrator-over-agent-plumbing` shape applied to scheduling.

   The operative word is **poll**, not *timer*. A fixed-schedule job whose output is the deliverable — a morning brief, a daily digest — always has something to say, so it is paying for output rather than for the privilege of checking. That case is legitimate when approved and bounded; see *When NOT to Use*. Read this point as banning the timer that wakes to *look*, not every timer. For the brief-shaped version, see `morning-briefing-pipeline.md`.

**Then a backstop, because the gates are behavioral.** A zero-LLM auditor reads the *agent's own scheduler store* (not the OS crontab) and flags any **enabled** job whose id isn't on the allowlist. Wire it into the session-start snapshot and the daily health check, and alert the operator on a flag. The gate prevents; the auditor catches the slip — and it closes the invisibility that let the leak run for days.

## Why It Works

- **Deterministic-first removes the whole failure class** — a script that makes no model calls has no token bill to leak and no silent loop to enter, so most of these jobs should never be LLM-driven at all
- **An allowlist entry makes "approved" a checkable fact** — the auditor can compare the running scheduler against it deterministically, which a verbal approval can never support
- **The local-model question, asked at creation, catches the cheap win while it's cheap** — you decide token-free-vs-paid once, up front, instead of discovering the bill later
- **Auditing the agent's own scheduler closes the blind spot** — the leak wasn't unnoticed because it was subtle; it was unnoticed because nothing looked at the store it lived in

## When to Use

- Any agent that can create its own recurring/scheduled work (cron jobs, routines, wake-timers) and spends metered tokens per run
- Fleets where an assistant sets up background monitoring, sync, or digest jobs on the operator's behalf
- Anywhere a shared API key means one job's runaway spend can rate-limit or spend-cap *other* jobs

## When NOT to Use

- Purely interactive agents with no self-scheduling — there's no standing charge to gate
- Zero-LLM scheduled scripts (the thing you're steering toward) — they need the ordinary cron hygiene, not this gate
- A single, operator-approved, self-terminating LLM job with a bounded run count and a real per-run purpose — that's the gate working, not a violation

## Watch-outs

- **Audit the right store — and re-check that answer after you follow this pattern.** Initially the OS crontab is the safe one; the agent's own scheduler (a JSON store, a task DB, a routines table) is where the token-spending jobs are. Enumerate *that*, and confirm your auditor's scope actually covers it — a guard that reads the wrong store reports "all clear" about a place the leak isn't
- **Adopting "fire on new work" moves the spend into the store your auditor ignores.** This is the pattern defeating its own backstop, so it needs saying plainly. Point 3 tells you to migrate LLM turns out of the agent scheduler and into deterministic OS-cron scripts that wake a model only on change. Do that and the OS crontab is *no longer* the harmless one — it now holds every token-spending job you moved, while the auditor still reads only the agent scheduler and still reports all-clear. Observed in the source project: a `surgery-doc-sync.py` system cron fires a real `hermes chat` turn on change, and the allowlist auditor (`audit-llm-crons.sh`, a pure read of `~/.hermes/cron/jobs.json`) cannot see it. The fix is to define the audited surface as **"anything that can invoke the model,"** not "the agent's scheduler" — enumerate both stores, and grep the OS crontab for invocations of the agent CLI, not just for its scheduler file. Otherwise the more faithfully you follow point 3, the blinder the backstop gets
- **Disabled ≠ deleted.** Pausing a leaking job stops the bleed but leaves it one toggle from resuming. The auditor should only flag *enabled* jobs (so a paused one isn't noise) — but a cleanup pass should still delete what's truly dead
- **A spend cap makes this a resilience issue, not just a cost one.** Model the blast radius: what else dies when this key hits its cap? If the answer is "the whole assistant," the leak is a P1, not a line item
- **The backstop alert is operator-actionable — route it accordingly.** It's billing/health, so it goes to the operator, not to end users (see `quiet-by-default-notifications`), and it fires on the flag, not on every poll
- **"Could a local model do it" has a real answer both ways.** Don't reflexively say no. Tool-heavy turns: usually no (payload too big for a small local model) → prefer the deterministic script. Judgment-only text: often yes → a local model zeroes the cost. The point is to *check*, per job
- **Local-inference runners speak the hosted SDK's wire format; the library name is not the discriminator.** The documented way to drive Ollama or LM Studio from Python is through the OpenAI or Anthropic SDK aimed at a local port: `OpenAI(base_url="http://localhost:1234/v1")` for LM Studio, `base_url="http://localhost:11434/v1"` for Ollama. So a detector keyed on "imports openai" will incorrectly flag FREE local inference as billed spend, and this gets worse as local-model routing becomes standard. The discriminator is not the library imported; it is the parsed hostname in an explicit `base_url` parameter. A bare `import openai` still means spend (defaults to `api.openai.com`), and keep the over-flag bias: a stray `localhost` string elsewhere in the file must not silence a real hosted client. The allowlist checker and any auditor must parse URLs and imports (not just grep for strings), per `grammar-parsing-over-text-matching.md`

## Adjacent Patterns

- `deterministic-orchestrator-over-agent-plumbing.md` — the general form: put the mechanical work in a script, wake a model only for the irreducible judgment. Scheduling is one instance
- `quiet-by-default-notifications.md` — the sibling discipline for the *notification* a scheduled job emits: fire on change with a cooldown, route operator alerts to the operator. Shares the "fire on change, not on a timer" core
- `detached-launch-from-timeboxed-agent.md` — the legitimate way to run background work: launch a bounded, self-terminating job, not a standing poll loop
- `wire-into-existing-flows.md` — why the auditor must be wired into an existing forcing function (session-start snapshot, daily check) or it sits dormant and the blind spot reopens
- `morning-briefing-pipeline.md` — the most common *legitimate* scheduled job, and the one users ask for by name. It assumes a deterministic render; if you let a model write the brief, this gate is the half it does not cover

## Source

The household-agent repo, 2026-07-21. The household agent (a Raspberry-Pi assistant) had been asked to help get a set of TV episodes downloaded for a family member. In good faith it created an hourly LLM cron to "check download progress and upload when complete." One episode's download was permanently stalled (a dead torrent swarm), so "complete" never became true and the loop never terminated. Every hour a fresh model session woke, ingested 150K–370K input tokens, saw the download wasn't done, output "[SILENT]", and exited — ~$4.45 overnight for zero useful output, and enough sustained spend to trip the provider's monthly spend cap, which then failed an unrelated daily sync job on the same key. The session-start snapshot never showed it because that snapshot enumerated the OS crontab, never the agent's own scheduler. A rule against "LLM crons for monitoring" already existed in a memory file and had bent under the concrete, sympathetic request. The durable fix was structural: deterministic-first as the default, a creation-time approval gate backed by an allowlist (with a required "could a local model do this for free?" step), and a zero-LLM auditor that reads the agent's scheduler and flags any enabled job not on the allowlist — wired into both the session snapshot and the daily health check so the job can never again run unseen.
