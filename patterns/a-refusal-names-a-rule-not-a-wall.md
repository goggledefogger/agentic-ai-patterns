---
type: pattern
date: "2026-08-22"
source: The household agent's property-report lane (the agent board #49, #52, #53) — three nights of instruction tuning against an agent that could not run the command at all
tags:
  - agents
  - debugging
  - permissions
  - evidence
  - anti-pattern
---

# A Refusal Names a Rule, Not a Wall

When an agent declines to do something, it tells you *why* — and that why is a hypothesis it formed about its own failure, delivered in the register of a policy citation. It reads like a decision because it is phrased like one. Underneath, the agent may simply have been unable to act, in a way it cannot see and therefore cannot report. Tune the instruction and you are editing the wrong layer, confidently, for as long as the wording keeps almost working.

## The Problem

An assistant's job on a board task was to text a person a link. It posted:

> Cannot complete delivery: missing state file `…property-<slug>.json`. Without a recorded requester and channel ID, sending an external message is prohibited under safety rules.

Every noun in that sentence is real. There is a state file. There is a requester convention. There *is* a safety rule about external messages. So the fix looked obvious and was pursued for two rounds: the file was keyed on the short link code while the task was keyed on the address slug, so the lookup missed — fix the key, then reword the consent so a missing file is not a veto.

The wording got better. Nothing was ever delivered.

The actual cause was in a config file the agent never mentioned, because nothing in its context describes it: `approvals.cron_mode: deny`. A session woken by cron has no human to approve a command, so the runtime denies **every** command it attempts. The agent had tried the send five times across four shapes — direct, twice with a `timeout` prefix, and once through a `python3 -c` subprocess — and each one timed out unapproved. It then did what any reasonable actor does with five silent failures: it reached for the most plausible rule it knew and reported that.

No phrasing of the consent rule would ever have worked. The instruction layer was never the layer.

## Why the wall stayed invisible

Two things hid it, and both are ordinary.

**The refusal was well-formed.** A confident, specific, correctly-cited reason does not read as a guess. Had it said "five commands timed out and I don't know why," the investigation would have started in the right place on night one.

**A human kept finishing the step by hand.** Each time the automation failed, someone sent the link themselves — and the send log therefore held a delivery that looked like proof the path worked. It wasn't: it had gone to a different person than the failed task named, an hour after that task blocked. Manual completion is the most effective mask an automated failure can have, because it makes the lane look ~90% working while the automated path has a 0% success rate. The give-away is in the details of the successful case — recipient, timing, who typed it — not in its existence.

## The Pattern

**Before tuning what an agent was told, prove it could have acted.** Capability first, instruction second. The check is cheap and it is not a code read: run the thing the agent said it did, in the same context it runs in, and look at what comes back. Non-interactive is a *different context* from your terminal — the same command with the same permissions can be allowed for you and denied for it.

Three questions that separate the layers:

- **Did it attempt the action, or reason about attempting it?** Logs show attempts; the refusal text shows conclusions. Five denied attempts and "prohibited under safety rules" are the same event described from inside and outside.
- **Does a comparable success exist, and who performed it?** A prior success in the record is only evidence if the same actor produced it, unassisted, in the same context.
- **What does the runtime say, as opposed to the agent?** The agent reports its model of the system. Config and source report the system.

**Then ask whether the step wants an agent at all.** A step with no judgment in it — take a name and a link, send it — gains nothing from a model in the middle and inherits everything that can go wrong with one: it can decline, it can misreport, it can close the task without acting, and here it could not execute at all. Moving that step into the deterministic caller removed four failure modes at once and cost nothing, because there was never a decision being made.

## When it applies

Any unattended agent: cron-woken sessions, CI runners, queue workers, sandboxed tool use, an agent under a permission or approval layer it did not author and cannot introspect. The tell is a refusal that cites policy while the logs show attempts — or a lane that only ever succeeds when a person happens to be watching.

## Related

- [[no-delivery-without-arrival-accounting]] — the other half: this pattern is why the send never happened, that one is why nobody found out
- [[a-summary-can-invert-the-policy]] — an agent's account of a rule is not the rule
- [[a-partial-read-proves-presence-not-absence]] — the same evidence error, applied to reading instead of acting

## Source

the agent board repo #49, #52, #53 and the household-agent repo PRs #259–#261, 2026-08-20 to 2026-08-22. Mechanism confirmed in `hermes_cli/config_defaults.py` (`cron_mode: deny` — "block the command and let the agent find another way") and `tools/approval.py` (a cron approval context returns `approved: False` with no human to ask), not inferred from the key name.
