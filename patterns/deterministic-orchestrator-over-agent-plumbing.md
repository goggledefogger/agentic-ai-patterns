---
type: pattern
date: "2026-06-23"
source: The household-agent repo (a household agent on a Raspberry Pi) — podcast→wiki pipeline, a script
tags:
  - agents
  - reliability
  - determinism
  - pipelines
  - claude-code
---

# Deterministic Orchestrator Over Agent Plumbing

When an agent has to redo the same multi-step mechanical plumbing on every request — and fails a *different* way each time — stop teaching it the steps and give it one deterministic script that does them. Keep the model in the loop only for the steps that genuinely need judgment. The recurring-failure signal is the tell: same task, novel breakage each time, means the work is mechanical and the variance is the agent improvising glue it shouldn't have to.

## The Problem

A skill or routing rule describes a pipeline as a sequence the agent executes by hand: fetch X, transform it, write file A with this frontmatter, validate, write file B citing A, commit through the wrapper. Every step is deterministic, but the agent re-derives the glue each turn — and LLMs are non-deterministic, so each run breaks somewhere new.

Concrete recurrence, the household-agent repo, a single 24-hour window (2026-06-22 → 06-23). The request was always the same: "save this podcast's insights to the wiki." It failed four different ways across three conversations:

1. **Stale path** — the routing rule had a hardcoded path to `analyze.py` that had moved; `FileNotFoundError`, then several turns spent hand-editing the routing file.
2. **Summary-as-source** — the agent wrote its own key-insights summary into the Layer-1 *raw* slot (which must be the verbatim transcript); a provenance guard rejected it; the agent looped trying to satisfy it.
3. **GC'd transcript** — the transcript landed in a temp file that was unlinked before the agent could read it back; it re-fetched, hit the same GC, re-fetched again.
4. **Frontmatter format** — the `sources:` field was written as bare strings instead of `- path:` mappings; the wrapper's parser silently produced zero sources and emitted a misleading "no sources" error. The agent burned **~15 turns** — trying absolute vs. relative paths, re-reading docs, finally reading the wrapper's source code — before it found the required shape, then gave up and asked for help.

None of these is a reasoning failure. Each is *plumbing* the agent should never have been holding. The variance — four distinct breakages for one fixed task — is the diagnostic: **a task that fails a new way every time is a task that should be code.**

## The Pattern

Collapse the brittle assembly into one deterministic orchestrator script. The agent's entire job becomes: recognize the trigger, run one command, relay the result.

Draw the line at *judgment*, not at *steps*:

- **Mechanical → script.** Path resolution, file naming, frontmatter assembly, ordering, idempotency, calling the validator/wrapper, error classification, retries. The script owns all of it and produces artifacts that pass downstream guards *by construction* — e.g. it writes the verbatim transcript into the raw with the exact `source:`/`verbatim_capture:` frontmatter the provenance guard requires, every time, so the guard can never fire on a well-formed run.
- **Judgment → keep in the model (or an LLM step *inside* the script).** Summarizing a transcript into insights is real model work — but it lives behind a stable interface (`analyze.py --format json`), so the orchestrator consumes structured output and the non-determinism is contained to where it belongs.

The orchestrator emits a machine-readable result line the agent relays, and classified exit codes (`0` saved, `2` retryable, `3`/`4` specific failures) so the agent's response is a lookup, not an interpretation.

### The routing rule shrinks to one line

Before: a multi-paragraph skill describing raw-then-synthesis layering, frontmatter shapes, wrapper invocation, and failure recovery — ~20-50% of which the model followed on a given run.
After: "podcast/audio URL → run `podcast-to-wiki.py "URL"`; relay the summary it prints; don't hand-build the files." The discipline moved from prose the model must internalize into code that executes the same way every time.

## Why It Works

- **Removes the variance surface.** The agent can't get the frontmatter wrong if it never writes the frontmatter.
- **Failures become reproducible.** A bug in a script is debuggable once and fixed for all future runs; a bug in agent improvisation recurs with new symptoms.
- **The guard becomes unreachable on the happy path.** When the producer is deterministic, well-formed output is guaranteed, so validators only ever fire on genuinely novel inputs — not on the agent's glue.
- **Cheaper and faster.** One tool call instead of a 15-turn flail.

## When to Use

- A task recurs and breaks differently each time (the headline signal).
- The steps are deterministic even though one or two sub-steps need the model.
- A downstream validator/guard keeps rejecting the agent's hand-built output.
- You catch yourself adding *more* prose to a skill to fix a behavior — that's a smell that the behavior wants to be code.

## When NOT to Use

- The task is genuinely one-shot or exploratory — scripting it is premature.
- Every step needs judgment (no mechanical core to extract).
- Inputs are too varied to enumerate; a rigid script would be wrong more often than the agent. Prefer the script when the *shape* is fixed and only the *content* varies.

## Anti-pattern: fix-the-prose

The reflex when an agent botches a multi-step task is to write a clearer instruction — a bolded warning, a counter-rationalization table, a "MANDATORY" header. That raises compliance from maybe 50% to maybe 70% and adds tokens every turn. If the step is deterministic, the correct fix is to remove the decision from the model entirely. Prose is for judgment; code is for plumbing. (Companion: `personality-as-discipline.md` covers the inverse case — behavioral rules that genuinely *can't* be scripted because they're about how the agent reasons; encode those as persona, not prose either.)

## How to Adopt

1. Find the recurrence. Scan recent transcripts/logs for a task that failed more than twice with *different* errors. That's your candidate.
2. List the steps. Mark each one `mechanical` or `judgment`.
3. Write the orchestrator covering every `mechanical` step end to end; shell out to a stable interface for each `judgment` step.
4. Make outputs pass downstream guards by construction — encode the exact contract the validator checks.
5. Add classified exit codes + a machine-readable success line.
6. Shrink the skill/routing rule to: trigger → one command → relay result → "don't hand-build this."
7. Test through the agent's real execution path, not just unit tests — the timeouts and permission gates only show up there (see `smoke-tests-with-real-data.md`).

## Adjacent Patterns

- **Detached launch from a time-boxed agent** (`detached-launch-from-timeboxed-agent.md`) — once the plumbing is one script, this is how the agent runs it when it's slower than a tool-call timeout.
- **Personality as discipline** (`personality-as-discipline.md`) — the complement: behaviors that can't be scripted because they're judgment get encoded as persona, not prose.
- **Wire adoptions into existing flows** (`wire-into-existing-flows.md`) — the orchestrator only helps if the routing rule actually points at it.
- **Smoke tests with real data** (`smoke-tests-with-real-data.md`) — prove the deterministic producer's output passes the real downstream guards.
- **Subagent ceremony by task type** (`subagent-ceremony-by-task-type.md`) — same mechanical-vs-judgment cut, applied to how much dispatch ceremony a task earns.

## Source

The household-agent repo, `scripts/podcast-to-wiki.py` + the 2026-06-22/06-23 podcast-ingestion incidents (summary-as-raw, GC'd transcript, `sources:` frontmatter loop). One deterministic orchestrator replaced a routing rule the agent had failed to execute four distinct ways in 24 hours.
