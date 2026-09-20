---
type: pattern
date: "2026-08-03"
source: The personal agent — walk-and-talk teardown, stale `tailscale serve` proxy found 2026-08-03 (ux-debt 42 + 66)
tags:
  - lifecycle
  - teardown
  - verification
  - shared-state
  - anti-pattern
---

# A Live Reference Outlives Its Owner, and Everyone Keeps Traversing It

When a process dies, the things it *held* die with it. The things it **registered elsewhere** do not. A proxy target, a port registration, a tunnel, a DNS entry, a webhook subscription — these live in a second system that never learned the first one was gone, and other components keep reading them and acting on them as if they were live.

The owning process's cleanup is the obvious place to fix this, and it is not sufficient — because cleanup frequently never runs at all.

## The Problem

`stale-pointer-asserts-confidently.md` covers a stale **fact** — a cached path in a registry that answers confidently and wrongly. This is the other half:

|  | `stale-pointer-asserts-confidently` | This pattern |
|---|---|---|
| What's stale | a *fact about* a resource, copied elsewhere | a *live reference* others still traverse |
| Who's harmed | whoever reads the fact and believes it | whoever follows the reference and gets nothing |
| Symptom | a confident, specific, wrong answer | a healthy-looking path that silently goes nowhere |
| Fixed by | resolving at use time, holding questions not answers | reconciling at start; owning your own target |

A stale fact misleads a reader. A stale live reference is worse: **every layer reports healthy**, because each one is. The reference exists. The second system is up. The DNS resolves. The certificate is valid. The only broken thing is the *edge* between the reference and the thing it points at — and that edge is usually the one the user physically traverses, so they are the component that discovers it.

Concrete instance (the personal agent, walk-and-talk, 2026-08-03). A voice bridge registers `tailscale serve` so the operator's phone can reach it over HTTPS. On teardown, a step exists to clear that registration. It ran, and the proxy survived — still advertising `macbook-16-town.ts.net -> 127.0.0.1:8790` with nothing listening on 8790. The step was written:

```bash
{ tailscale serve --https=443 off >/dev/null 2>&1 || tailscale serve reset >/dev/null 2>&1; } \
  && echo "• reset the Tailscale HTTPS proxy" || true
```

Both attempts suppressed, chain terminated in `|| true`, no read-back. Run by hand afterward the command works and returns 0 — **the command was never the bug.** The bug was that no caller could distinguish "worked" from "did nothing," so the orchestrating agent reported success from having *invoked* the step.

It was the second occurrence. Two months of plans had already adopted the right-sounding rule — *every background component registers a teardown* — and the proxy step complied with it both times. It was registered. It ran.

## The Pattern

1. **Reconcile at start against observed reality, not against a flag.** The owning system recorded `voice_mode: false` — an accurate statement that the last session ended cleanly — while the proxy it had registered was still up. The recovery path consulted the flag, concluded there was nothing to reconcile, and skipped. **A flag describes what you believe happened; only a read describes what is.**

2. **Assume teardown never runs.** Crashes, hung tasks, OOM kills, and `kill -9` all bypass it. In the instance above the session was ultimately closed with `tmux kill-session`, so cleanup did not execute at all — a perfectly hardened teardown would have been bypassed. Clean shutdown is a courtesy. **Convergent start is the contract.**

3. **Every death proves it died.** Registering a teardown step is not verification; the step must read the state back and report *that*. An exit code certifies that a command ran, not that a resource is gone (`opaque-write-needs-a-read-back.md`). A step whose success is reported to a human may not suppress its output or swallow its failure.

4. **Own your own target at start.** A component that registers shared external state should point it at *its own* resource on startup rather than inheriting whatever a predecessor left. This makes the stale case self-healing on the next run instead of requiring the previous run to have been polite.

5. **Verify the edge the user traverses, not the one you can reach.** `127.0.0.1:<port>/status` returning 200 proves the process is alive and proves nothing about the path the phone takes. Fetch through the public URL before advertising it.

## Why It Works

- **Reconciliation has no dependency on the past.** A start that reads reality is correct after a clean exit, a crash, a kill, and a first-ever run — one code path, no history to trust.
- **Read-back collapses the gap between intent and outcome.** It is the only mechanism that catches a command that succeeds while achieving nothing, which is exactly the failure that survives every "did you register a teardown?" review.
- **Self-owned targets bound the blast radius to one run.** If each start repoints its own reference, a stale one can survive at most until the next start.

## Watch-outs

- **The rule you already have may be the wrong rule, stated confidently.** "Every birth has a death" was adopted, restated across two plans, and complied with — and did not prevent either occurrence. When a failure recurs under an existing rule, the fix is rarely a third restatement; find the *unstated* half. Here it was: **every death proves it died.**
- **Registration reviews create false confidence.** "Is it in the teardown funnel?" is answerable, auditable, and satisfied by code that does nothing.
- **A guard scoped to a remembered range silently skips.** The instance's check matched only ports 8780-8799; a bridge outside that window would have made the whole branch a no-op with no signal. Parse the structured state, don't pattern-match a remembered slice of it (`grammar-parsing-over-text-matching.md`).

- **The fix will reintroduce the same flaw one line away.** This is the sharpest thing the instance taught, and it happened under an explicit instruction not to. The change that removed the hardcoded port range from the proxy check wrote a *new* hardcoded port for the bridge check immediately below it — and the port it picked was not the one the incident occurred on, so the repaired sweep would still have missed the original bug. The instruction was followed where it was written and violated where it was merely implied. When you remove a class of flaw, grep the diff for the class, not the line.

- **A verifier earns trust by being exercised, never by being read.** The same sweep shipped with a process pattern that matched a *linger babysitter* instead of the process it named, and an invented JSON schema that returned the correct verdict for the wrong reason — so it looked right, reported right, and could not actually see. Both died on the first real run. Corollaries: a check may not hide its own stderr (a bare `except` plus `2>/dev/null` makes a bug *in the validator* indistinguishable from the failure it reports), and a resource the teardown deliberately leaves running is not a survivor — counting a warm component as a stray fires the alarm on every start and thereby retires it.
- **The agent reporting the outcome is part of the system.** If the only record is prose the agent wrote from intent, the failure is unobservable no matter how good the script is. Give teardown a structured log and require the report to cite it (`log-the-verdict-not-the-volume.md`).

## Adjacent Patterns

- `stale-pointer-asserts-confidently.md` — the stale-*fact* half of the same hazard; this is the stale-*reference* half.
- `convergent-standup-sloppy-quits.md` — the positive discipline: converge at start rather than depend on a clean quit.
- `teardown-verified-startup-never-written.md` — birth/death symmetry, the rule that was already adopted here and proved insufficient alone.
- `opaque-write-needs-a-read-back.md` — why an exit code cannot certify a postcondition.
- `health-check-that-never-exercises.md` — each layer reporting healthy while the composition is broken.
- `log-the-verdict-not-the-volume.md` — the structured record that lets a reporting agent cite instead of narrate.

## Source

The personal agent `walk-and-talk` skill, `scripts/session.sh` `cleanup_all()` and the `start` recovery path. Stale `tailscale serve` proxy observed live 2026-08-03 (bridge dead on :8790, proxy still advertising, `voice_mode: false`); first occurrence 2026-08-01 as ux-debt 42, second written up as ux-debt 66. Fix planned as `epics-walk-teardown-2026-08-03`.
