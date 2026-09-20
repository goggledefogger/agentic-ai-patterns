---
type: pattern
date: "2026-07-29"
source: Evaluating a third-party "token saver" skill downloaded as a zip (2026-07-29). Tested its selector against a planted needle in a 700-line transcript, then compared its claims to the ponytail plugin's published cost benchmark at `~/.claude/plugins/cache/ponytail/*/benchmarks/results/2026-06-17-cost-verification.md`.
tags:
  - token-efficiency
  - skills
  - adoption
  - verification
  - claude-code
---

# The Useful Core — Adopt a Skill's Rules, Not Its Machinery

A downloaded efficiency skill usually arrives as two things wearing one name: a short list of behavioral rules, and a pile of machinery that claims to enforce them. The rules are portable and often good. The machinery is where it fails, and it fails quietly, because a retrieval step that finds nothing and a retrieval step that ran correctly produce the same shape of output. Extract the core, delete the rest, and make the deletion a habit rather than a one-time cleanup.

## The Useful Core

Three rules survived a real audit of an 8-strategy token-saving skill. Everything else was either already covered elsewhere or actively wrong.

1. **Read passages, not files.** Search first, then read the span you need. When the first bounded read misses, take a second bounded read. Never fall back to loading everything, because that is the move the discipline exists to prevent
2. **Count the whole job, not the turn.** Every call counts: planning, selecting, answering, verifying, repairing. Moving work to a subagent, a cheaper model, or another host is not a saving unless the combined total falls. When a host does not report a number, write `unavailable` and do not infer one
3. **Never retry a token or usage limit failure.** A limit error is not transient. Reduce the input, pick a cheaper path, or wait for the reset. Retrying converts one refusal into several

The other 5 strategies were already handled: output brevity and artifact size by the ponytail ruleset, model choice by a model-routing skill, and measurement by a token-efficiency audit. Overlap is the normal case when adopting, so check what you already run before you install anything.

## What the Machinery Got Wrong

The skill shipped a local passage selector, which is the part that looked like real engineering. Three failures, all found by planting one decision line in a 700-line synthetic transcript and asking for it back.

| Test | Result |
|---|---|
| Question paraphrased ("Wednesday" for a line reading "mid-week sync") | 0 passages selected, 474-byte empty packet, reported as a successful selection |
| Question using the file's exact wording | Found it |
| "Summarize this transcript" | Returned lines 1-125 plus the tail, missing the planted decision |

The first failure is the dangerous one. Chunks scoring below a floor were dropped entirely, so a vocabulary mismatch produced an empty packet with a success message on stderr. The third is worse in practice: any request containing a broad verb (summarize, review, explain, analyze, edit) disabled the relevance filter and left file position as the only signal, so selection degraded to `head` while still reporting a passage count. Truncation labeled as selection is how a confident wrong summary gets made.

There was also a structural mistake specific to agent harnesses. Writing a 12KB packet to disk and reading it back costs more than a scoped `rg -C 5`, because the agent doing the reading is the model you were trying to protect. Packet building only pays off when the packet crosses into a different context, and the skill never wired that path.

## The Adoption Check

Before adopting any skill that claims a saving, spend 10 minutes:

- **Plant a needle and paraphrase the question.** Retrieval that only works when you already know the file's wording is a slower `grep`. This is a negative control, the run has to be able to fail
- **Try the boring verbs.** Summarize, review, explain. These are the most common asks and the most likely to bypass a relevance filter
- **Check the cost direction, not just the magnitude.** An always-on ruleset is re-sent as input on every call. Ponytail's own benchmark shows its 42-75% Claude saving reverses to 26-39% *more expensive* on reasoning models for exactly this reason
- **Prefer the thing with receipts.** A measured range with its method published, and a headline the authors corrected downward against their own data, outranks any asserted percentage

## Why It Works

- **It separates the transferable part from the fragile part.** Rules survive a harness change, a retrieval heuristic does not
- **It puts the burden on the claim.** A skill that reduces spend can prove it on 5 tasks in an afternoon, so absence of a measurement is itself a finding
- **It keeps the catalog small.** Three rules folded into an existing skill beats a new skill with 3 scripts to maintain, and the scripts were the broken part anyway

## When to Use

- Someone recommends a skill, plugin, or ruleset that promises lower cost or fewer tokens
- A long session is expensive and the instinct is to install something rather than change what gets read
- You are writing an efficiency skill yourself and reaching for a retrieval algorithm

## When NOT to Use

- The saving is a hard platform mechanic rather than a behavioral claim (prompt caching, a batch endpoint, a smaller context window). Read the provider docs, no audit needed
- The skill ships its own benchmark with a reproduction command. Run that first, and only audit if it fails to reproduce

## Watch-outs

- **An efficiency skill is itself input.** A 7KB ruleset loaded to tell you to load less is a real cost, and it cannot undo the turn that loaded it. Keep the core short enough that its own footprint is not the problem
- **Silent empty output is the failure mode to test for, not the crash.** Scripts that report "selected 0 passages" and exit 0 will be trusted by the next agent that runs them
- **A skill can be safe and still be wrong.** The one audited here had no network calls, skipped `.env` and secret-named files, and escaped injection markers in its output. Careful authorship says nothing about whether the algorithm works

## Adjacent Patterns

- `verify-adoption-against-installed-source.md` — the same lesson one level up, grep the artifact you actually installed before adopting a plan's claim about it
- `verification-needs-a-negative-control.md` — why the planted-needle test needs a paraphrase leg that must fail
- `unrun-checks-read-as-passing.md` — "did not run" and "ran and passed" share a shape, which is what makes an empty packet dangerous
- `tdd-for-skills.md` — RED-phase a skill against pressure scenarios before shipping it

## Source

- `~/Downloads/token-saver-skill.zip`, a third-party token-saving skill, audited 2026-07-29. Safe, carefully written, and wrong in the selector
- `~/.claude/plugins/cache/ponytail/*/benchmarks/results/2026-06-17-cost-verification.md`, the measured comparison, 30 pooled reps across Haiku, Sonnet, and Opus, with the cross-provider reversal
