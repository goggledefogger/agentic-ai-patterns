---
type: pattern
date: "2026-08-01"
source: The household-agent repo / the household agent (Pi agent) — the 2026-07-31 vendor-list drift incident
tags:
  - agents
  - drift
  - config-as-data
  - guard-design
---

# Give the Agent a Lawful Write Surface

When an agent keeps violating a "never edit X" rule, audit whether a lawful surface exists at all before treating it as a discipline problem. Usually the rule is asking the agent to drop real work on the floor.

## The Pattern

The diagnostic comes first, and it is one question: **what was the only implementable form of that request?**

The household agent was asked to add two brands to a daily price tracker. The tracked list was a Python literal inside the script; the script's only external read was its price-history state file. There was no vendor data file, no config, no override. Adding a vendor *was* editing the script, and the script lives on the Pi. She edited it on the Pi. The repo has said "NEVER edit directly on the Pi" the entire time, and it has never once held — because on that request, obeying it meant refusing the work.

So don't strengthen the rule. Add the surface. Split every agent-tunable list into two halves:

```
repo/scripts/battery-vendors.json          SEED — ships with the repo, deploy OVERWRITES it
~/.openclaw/state/battery-vendors.json     OVERLAY — machine-local, never deployed
```

Merge them by a stable key with the overlay winning, and give the agent a **deterministic writer** rather than a prose instruction:

```bash
check-battery-prices.py --add-vendor --model "Acme PS-3800" \
    --vendor "Acme Direct" --url "<product page>" --match "PS-3800" --wh 3840
```

Four properties do the actual work:

- **The overlay must survive a deploy.** Verify it, don't assume it — grep the deploy script for every path it writes under your state directory. Ours wrote exactly one file there, which is the difference between a durable surface and a slower drift.
- **The writer enforces what the doc merely requested.** The docs had asked for "only URLs verified to resolve get added" for months. `--add-vendor` probes the page and refuses anything it can't read a live price from, so a bad URL fails at add-time instead of silently reporting UNKNOWN every morning.
- **A missing seed degrades to a stale list, never to zero.** Zero vendors renders as a clean run with nothing to report — a blind check that reads as a healthy one.
- **An unreadable overlay warns loudly.** It must never pass for "no additions," or the agent's work vanishes into a silence that looks like success.

For changes that are genuinely *code* and not data, the same logic applies one level up: the agent needs somewhere to put the proposal — a queue file, an issue, a branch — or "don't edit code" still means "discard your finding."

## Why It Works

Rules are a tax on every turn and they bend under pressure; a missing surface bends the agent every single time, deterministically. If the correct action is also the only easy action, compliance stops being a behavioral question.

It also explains the recidivism honestly. The edits in this incident were *good* — every price matched a live probe to the cent, the ledger sync ran clean and deduped, the existing test suite still passed. Nothing about that is a discipline failure, and a stricter rule would have suppressed competent work rather than routed it.

The trap to watch for is precedent-blindness. The exact pattern already existed **one directory away, in the same skill**: a notebook registry doing seed-plus-local-overlay, with the deploy-overwrites rationale spelled out in a comment. Nobody generalized it, because the search everyone runs is for the noun ("vendor config") rather than the shape ("thing the agent needs to write that a deploy would clobber"). Grep for the shape.

## When to Use

- An agent has broken the same "don't touch X" rule more than once, especially when its edits were competent
- You're about to write a stricter version of a rule that already exists
- A deploy overwrites files the agent is expected to keep current
- Any hardcoded list inside a script that a human or agent is expected to extend

Don't reach for it when the agent's edits are genuinely wrong, or when the change truly needs review before it takes effect — that's the proposal-queue half, not the overlay half.

## Source

the household-agent repo — `scripts/check-battery-prices.py` (`load_vendors`, `add_vendor`, `write_local_vendors`) and `scripts/battery-vendors.json`, 2026-08-01. Prior art it should have copied from the start: `skills/shared/notebooklm/scripts/ask.py:35` `load_registry()`. Incident and the parked plan it revives: `_bmad-output/planning-artifacts/plan-2026-08-01-agent-write-surfaces-runtime-overlays-and-proposal-queue.md`, `docs/fleet-script-ownership-proposal.md` (PARKED 2026-04-10).

Related: `registry-based-monitoring.md` (the registry shape), `memory-substrate-selection.md` (structured truth plus derived view), `sensitivity-tiered-access-control.md` and `deterministic-orchestrator-over-agent-plumbing.md` (why the rule wasn't the fix), `unattended-run-discipline.md` (the boundary this pattern deliberately softens for *data*, and keeps for code).

## Addendum (2026-08-17): the named verb, and discovery as part of the surface

A second live case sharpened the pattern. A save ritual hit a store whose
writes are gated (no shell for workers, a deny hook on the path), and the
right move was not a widened permission but a **named verb**: one fixed
script per action (save_brain.py beside the earlier render_report.py) - the
command is fixed in the file rather than chosen by a model, the target
resolves by logical name from the location map, it cannot do the adjacent
thing (save commits, never pushes), and contents never enter the caller's
context. Each new action a gated store needs gets its own named verb; the
verb catalog IS the write surface.

And the sharpest lesson came from the human's reaction: "I shouldn't need
to say that." The ritual initially required the user to name which verb
their own plumbing uses - which is the surface failing at discovery. So:
**discovery is part of the surface.** A caller must find the verbs itself
(a conventional location like scripts/, the store's own docs) before ever
asking, and only propose building a new verb when none exists - with
consent, never silently. A lawful surface nobody can find still reads, from
the outside, exactly like a missing one.

