---
type: pattern
date: "2026-10-09"
source: A local-model lane refused to start a server unless 60% of memory was free; a test brief told the worker to set that to 10 so the run would pass, and the 8 GB laptop ran out of memory
tags:
  - agents
  - guards
  - local-models
  - anti-pattern
  - reliability
---

# A Guard You Lower Is Not a Guard

A tool that runs a local model has a memory guard: it refuses to start a model server unless a set share of memory is free. The default was 60%. During a test run on an 8 GB laptop, the brief handed to the worker said to set it to 10, so the run would succeed.

It did. A 3.9 GB model at a 16K context answered end to end: the first chat took 77 seconds, the follow-up 4.5. Free memory went from 42% to 8%. Swap went from 2.4 GB to 4.3 GB. The operating system showed the person an out-of-memory warning. The guard did its job right up to the moment someone with a goal turned it down.

## Why the wrong move was attractive

The guard was blocking the one thing the run existed to show. A floor of 60% free is conservative, so lowering it looked like a calibration, not a bypass. And the run did pass, which made the lowered number look vindicated. A result you wanted is poor evidence that the limit was wrong.

## The rule

**A threshold that can be tuned at run time to get a result is a preference, not a protection.**

The shape that holds is a budget computed from the thing being loaded, checked against what the machine has, and decided before the action:

1. **Need:** the file size plus a working allowance for the context window.
2. **Have:** free memory minus a reserve for the operating system and the person's other work.
3. **Decide once, before loading.** If need exceeds have, do not start. Never relax the numbers because the result is wanted.
4. **Refuse with an alternative.** The honest output is "not this, but here is what would": a smaller model, a smaller window, or "not on this machine".

## When you see this

Any limit someone is tempted to lower mid-task to make the run go: memory floors, disk floors, rate limits, timeouts, retry caps, "just this once" overrides. Especially an agent brief that says "set the guard low so it passes".

## Why it matters for agents

An agent with a goal treats every guard as an obstacle unless the guard has no knob. Put the budget in code the agent runs but cannot edit at run time, and make the refusal name the alternative so the agent has somewhere to go besides around.

**The tell:** a guard whose threshold is a setting the requester of the result can reach. Ask who can change the number, and when.

## Adjacent Patterns

- [A Guard's Evidence Must Outlive the Failure It Detects](guard-evidence-outlives-the-failure.md): the evidence side of the same discipline.
- [Local-Model Agentic Tool-Calling](local-model-agentic-tool-calling.md): the setup this run was exercising.
