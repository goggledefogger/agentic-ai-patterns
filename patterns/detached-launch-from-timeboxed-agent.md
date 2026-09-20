---
type: pattern
date: "2026-06-23"
source: The household-agent repo (a household agent on a Raspberry Pi) — a script --background --notify;
  second incident 2026-08-20 — the personal agent scripts/property_runner.py, harness tearing down background tasks
tags:
  - agents
  - reliability
  - automation
  - claude-code
---

# Detached Launch From a Time-Boxed Agent

When an interactive agent must kick off work that runs longer than its own tool-call timeout, it can't run the work inline and it can't just give up — a human is waiting. The job has to self-detach, return control to the agent in under a second, and report back out-of-band when it finishes. Two non-obvious constraints make or break this: the launch command must stay in the shape the agent's permission layer already allows, and the report-back can't depend on the agent still being in the loop.

## The Problem

Agent tool calls are time-boxed. A Claude-Code-style `terminal`/`execute` tool typically kills a command at a fixed deadline (60s in the case observed). Plenty of legitimate work runs longer: fetching a transcript through a fallback waterfall, a build, a large download, anything with a slow network tier.

Run that work inline and the tool call is killed at the deadline — and because it was killed mid-flight, **nothing is saved.** Worse, the agent often *believes* it succeeded: it had already composed "on it, I'll let you know when it's done," the subprocess died when the turn ended, and no mechanism existed behind that promise.

The household-agent repo, 2026-06-23. A user sent a podcast link; the routing rule (correctly) had the agent run the ingestion pipeline. The transcript fetch took longer than 60s. The `terminal` tool returned `exit_code 124, "Command timed out after 60s"`. The agent replied, accurately-sounding, "that's taking a bit longer than usual to fetch the transcript — I'll let you know when it's ready and saved to the wiki." The process was dead. Nothing was written. There was no "later."

The obvious fix — background it with `setsid nohup … &` — introduced a *second*, sharper failure. That compound shell line couldn't be matched against the agent's command allowlist (the plain `python3 script.py "URL"` form was auto-approved; `setsid bash -c '…' &` was not). It was held for confirmation, timed out after 120s with no human to approve it, and was **denied**. The agent retried several times, every attempt blocked, then gave up. So: inline gets killed and saves nothing; the naive background wrapper gets denied and runs nothing. (This is the interactive mirror of `unattended-run-discipline.md`'s "compound shell defeats allowlists" — same root cause, opposite context.)

## The second reaper: the harness itself, at an unpredictable moment

A timeout is not the only thing that kills long work, and the harness's *own* background-task facility is not a safe harbor. The personal agent, 2026-08-20, re-running a property-research pipeline: three jobs launched as Claude Code background tasks (the sanctioned long-running mechanism, not an inline call) were killed harness-side. Twice. The first attempt died about two minutes in, the second about thirty — same three jobs, same command, wildly different lifetimes, no error and no partial output either time.

**The first hypothesis was a concurrency limit, and it was wrong** — worth recording, because the wrong guess is instructive. Three had died twice while one and two ran clean, so concurrency looked causal; it was a coincidence of *when interruptions happened*. No concurrency limit is documented anywhere, and the arbitrary lifetimes never fit it. The real explanation is a family of known defects in which the harness tears down background work on ordinary events: `anthropics/claude-code` #21167 (pressing ESC kills ALL background tasks), #50572 (background shells terminated on turn end), #63023 (silent death on session pause/resume), #32052 (`/clear` kills running agents), #28875 (an agent stopping a long job on its own). Eight such issues are closed with no fix; the facility is documented for "dev servers or watch builds" and is simply not dependable for 30–60 minute work at any concurrency.

**Generalize past the specific trigger.** Chasing "was it ESC or was it three?" is the trap — the durable lesson is that *the harness may end your background work at any moment for reasons you cannot see, and will not tell you it did.* Every instinct a timeout teaches you is wrong here:

- **There is no limit to raise.** With a timeout there is a number to find and extend (this same pipeline had a real one, raised 40m→60m the day before, correctly, after reading the timeline). Here there is no number anywhere — not in your code, not in the docs.
- **The kill is silent and total.** Status came back `killed`, exit code absent, log files holding nothing but the start line. Thirty minutes of research left no partial artifact — indistinguishable from a job that never started.
- **Retrying identically is the trap.** The second attempt was a good-faith relaunch that cost another thirty minutes and taught nothing new. Two data points with different lifetimes is the signal to stop retrying and change the *ownership* of the work — not to theorize harder about the trigger.

Rule out resources before blaming the harness, because they present alike: check the OS for actual pressure (on macOS, `log show --predicate 'eventMessage CONTAINS "memorystatus"'` plus `memory_pressure`). Here the machine sat at 51% free with no jetsam kill of any job — which is what promoted "the harness reaped it" from a guess to a finding.

The fix is the same instinct as the rest of this pattern, one level up: **stop letting the harness own the work.** `nohup … & disown` into a log file makes the job an ordinary OS process that no session supervisor can reap, and a separate watcher tails the logs for outcomes. The self-daemonizing `--background` flag below is the better long-term shape; `nohup + disown` is its one-line field expedient when you are mid-incident and the pipeline has no such flag yet.

And write the launch shape down where the next launcher reads it. A finding like this costs an hour per rediscovery, and the person who rediscovers it is usually you, next week, in a fresh session with none of this context.

## The Pattern

Push the backgrounding *inside the tool* so the agent-facing command stays in the already-allowed shape, and make the job report back through a channel that doesn't need the agent.

### 1. Self-daemonize behind a flag

Give the long-running script a `--background` flag that double-forks and returns immediately:

```python
def _daemonize(log_path):
    # single-threaded, no subprocess spawned yet → fork is safe here
    if os.fork() > 0:
        return False              # launcher: caller returns 0 to the agent now
    os.setsid()
    if os.fork() > 0:
        os._exit(0)              # session leader exits; worker can't reacquire a tty
    fd = os.open(log_path, os.O_WRONLY | os.O_CREAT | os.O_APPEND, 0o644)
    os.dup2(fd, 1); os.dup2(fd, 2)
    os.dup2(os.open(os.devnull, os.O_RDONLY), 0)
    return True                   # worker: run the long pipeline
```

The agent runs `python3 my-pipeline.py "URL" --background --notify` — **the same `python3 …` shape the allowlist already passes** — and the launcher returns in well under a second. The detach is an implementation detail the permission layer never sees. `setsid` + the second fork put the worker in its own session, so it survives the tool reaping the launcher's process group.

Validate fast/synchronous prerequisites *before* the fork (so a misconfig still surfaces to the agent), then daemonize *before* the long work.

### 2. Report back out-of-band

The worker is detached; the agent's turn is long over. So the worker notifies the user itself through a no-LLM side channel — a Telegram/Slack/webhook ping, an atomic write to a watched status file (`filesystem-queue.md`), an email. Success carries the result; failure carries the *actionable reason*, pre-worded for the user ("couldn't get a transcript yet — retryable" vs. "a guard rejected the write"). The agent never polls, waits, or re-runs.

### 3. Teach the agent the launch and the boundary

The routing rule says: run the one allowed command, then tell the user you're on it and you'll ping them — and **if the command is ever blocked, do not escalate to `setsid`/`nohup`/`&` wrappers** (those get denied); report the block instead. The escape hatch the agent reaches for under pressure is exactly the one that fails.

## Why It Works

- **The command stays allow-listed.** No compound shell, so no confirmation prompt, so no denial.
- **The agent's turn stays cheap.** Sub-second launcher return; the agent answers the human immediately and moves on.
- **The promise has a mechanism.** "I'll let you know" is backed by the worker's own notify, not by an agent that's already gone.
- **Failures are legible.** Classified outcomes become pre-worded user messages instead of silence.

## When to Use

- Agent-triggered work can exceed the tool-call timeout (network waterfalls, builds, transcoding, large fetches).
- A human is waiting, so "log and stop" (the unattended answer) isn't acceptable — the work must actually proceed.
- The runtime has a command allowlist that treats compound shell differently from a plain invocation.

## When NOT to Use

- The work reliably finishes inside the tool timeout — just run it inline and relay the result directly; async-notify is needless latency and indirection.
- It's an unattended/scheduled run with no waiting human — there, prefer fail-loud-and-stop (`unattended-run-discipline.md`), not fire-and-forget.
- You have a real job runner/queue already — enqueue into it instead of hand-rolling a daemon.

## Anti-pattern: the promise with no mechanism

The most dangerous shape isn't a visible error — it's the agent that *says* "I'll follow up when it's done" after launching something that was killed or never started. It reads as success to the user and to the agent's own transcript. If your agent can emit a follow-up promise, make sure a real out-of-band channel backs every path that promise can take, including failure.

## Anti-pattern: escalating the wrapper to beat the allowlist

When the plain command is blocked, the instinct is a beefier shell line (`setsid`, `nohup`, `&`, `$(...)`). That's the wrong direction — the allowlist is information. Keep the agent-facing command minimal and move complexity *inside* the tool; if even the plain command is blocked, that's a finding to surface, not to engineer around (same lesson as `unattended-run-discipline.md`, from the interactive side).

## How to Adopt

1. Find agent-launched commands that can exceed the tool timeout (grep logs for timeout/exit-124).
2. Add a `--background` self-daemonize flag (double-fork + `setsid` + stdio→logfile). Validate prerequisites before the fork.
3. Add an out-of-band `--notify` path that fires on every terminal outcome with a pre-worded, user-facing message.
4. Update the routing rule to the plain `cmd … --background --notify` shape + the no-escalation boundary.
5. Verify through the *real* agent path — confirm the launcher returns fast, the command isn't held for confirmation, the worker completes detached, and the notification lands (`smoke-tests-with-real-data.md`). Unit tests won't reveal the timeout or the permission denial; only driving the live agent does.

## Adjacent Patterns

- **Unattended-run discipline** (`unattended-run-discipline.md`) — the sibling for the *no-human* case: there the answer is fail-loud-and-stop; here a human is waiting so the work must proceed via detach-and-notify. Same allowlist root cause.
- **Deterministic orchestrator over agent plumbing** (`deterministic-orchestrator-over-agent-plumbing.md`) — `--background` is most useful once the long work is already one deterministic script.
- **Filesystem queue** (`filesystem-queue.md`) — an atomic status-file write is a clean out-of-band report-back channel for the detached worker.
- **Wire adoptions into existing flows** (`wire-into-existing-flows.md`) — the launch command has to be the thing the routing rule actually invokes.

## Source

The household-agent repo, `scripts/podcast-to-wiki.py` (`--background` / `--notify`) + the 2026-06-23 incident: inline run killed at 60s (saved nothing); `setsid` wrapper denied by the allowlist (ran nothing); self-daemonize in the allowed command shape fixed both.

Second incident, the personal agent `scripts/property_runner.py`, 2026-08-20: three property-research jobs launched as Claude Code *background tasks* were torn down harness-side twice — at ~2 and ~30 minutes — silently, with 51% free memory and no jetsam kill. A concurrency limit was the first hypothesis and did not survive contact with the issue tracker (#21167, #50572, #63023 and five siblings, mostly closed unfixed). Fixed by giving the pipeline its own `--background` flag (double-fork + setsid) rather than a `nohup … & disown` each caller must remember, plus a log-tailing watcher for outcomes.
