---
type: pattern
date: "2026-05-01"
source: A course participant's home-agent system (the workspace repo)
tags:
  - skills
  - anti-hallucination
  - verification
---

# System-Understanding Protocol

A meta-skill that enforces grounded, cited responses about system state. Designed to prevent the dominant failure mode where an agent reads a few files and presents the blended result with total confidence — no staleness flags, no separation between what was read and what was inferred.

## The Problem

Most "be careful, cite your sources" prompts collapse under pressure. The model rationalizes ("I'm pretty sure", "the README probably says", "I read a related file so this specific detail must be true") and produces a confident-sounding answer that mixes real and fabricated detail. A reader can't tell which sentences are grounded and which are synthesis.

The failure is visible in baseline tests: a clean subagent asked "how does the memory pipeline work?" produced specific plist labels, cron schedules, file paths, and consumer counts — *and claimed those facts came from a STATUS.md it had read*. They came from no file. The reader walks away with a plausible picture that is part fact, part fabrication, with no way to tell them apart.

## The Pattern

A 100–150 line meta-skill (`SKILL.md` only, no scripts) with five components:

### 1. Four-pass protocol

Stop at the first pass that answers with citable confidence. Each pass is more expensive than the last.

1. **Aggregate pass** — read the curated summary docs (README, STATUS, runtime-rules, fund-context, product-brief). Always check the `_Generated_` header. Flag if stale.
2. **Source pass** — read the actual source under `skills/`, `automation/`, and any sibling repos. Read the SKILL.md, not a description of it.
3. **GitHub pass** — `gh repo list`, `gh repo view`, `gh api .../commits`, `gh pr list`, `gh issue list` for repo-level state.
4. **Live state pass** — `launchctl list`, `git status`, `git log`, MCP config, SSH-to-runtime-host if available. Flag explicitly when the runtime host isn't reachable.

### 2. Atomic citations

Every factual claim names the file path, command, or repo consulted. **Atomic = `file:line` or `file:section-heading`**, not just file-path. Coarse citations let fabrication hide inside "I read the README." Tag inline so uncited specifics stand out.

### 3. Staleness flags

If the aggregate doc's header timestamp is >24h old or generated from a different host, say so explicitly: *"BUILD-STATUS.md was regenerated 43 hours ago from mac.lan. Treat live-state numbers as approximate."* Don't paper over staleness; advertise it.

### 4. Rationalization table

The piece that makes the skill stick under pressure. Names the common excuses by quote and counters them.

| Excuse | Reality |
|--------|---------|
| "I'm pretty sure X is the case" | Check. Memory is how hallucinations get cited back as fact. |
| "The aggregate doc has a summary, that's good enough" | Read the source when the question is about ground truth. |
| "Caller wants a quick answer" | Cited > fast. A wrong quick answer costs more than a slow correct one. |
| "I read a related file, so this specific detail must be true" | Cite only what you actually read. If a specific isn't in the cited text, it's synthesis, not citation. |

### 5. Red flags — STOP

The dominant failure mode, named explicitly:

> **Any specific number, name, date, or path that didn't come from a file you opened or a command you ran *this call*.**

Plus the soft-language flags: "I think X..." (without citation), "Probably running..." (without checking), "According to the README..." (without rereading).

## Output contract

Every response must:
1. Cite sources atomically (`file:line`, not `file`)
2. Flag staleness on aggregate-doc-derived claims
3. Name unknowns explicitly ("I can't confirm without SSH to the Mini")
4. No synthesis beyond cited material
5. Separate "what I read" from "what I inferred" — mark inferences

## Hard limits

- No writes, commits, deploys.
- No modifying launchd, killing processes, rewriting `.mcp.json`.
- If the caller asks for a *change*, refuse and name the correct escalation path.

## When to Use

Wire this skill into:
- **Onboarding agents** that answer "what's running here?" for new team members
- **Pre-action verification** before any agent acts on a doc-derived claim
- **Status/audit skills** that produce reports about live system state

Specifically: any time an agent's output will be acted on as fact about the running system, this protocol should run first.

## Example from the participant's system

`skills/meta/system-understanding/SKILL.md` — 120 lines, no scripts. The inventory is the filesystem; the skill is the protocol. Wired into `Explainer` (the onboarding agent) as a required sub-skill. Built specifically after a pressure-scenario test (`docs/superpowers/specs/2026-04-22-explainer-baseline-behaviors.md` S3) caught a clean subagent fabricating plist labels and cron schedules. The skill ships *because of* a documented failure mode, not as preventive hygiene.

## Adjacent Patterns

- **TDD-for-skills** (`tdd-for-skills.md`) — write the pressure scenarios that justify this skill *before* shipping it. The S3 fabrication is what justified system-understanding's existence.
- **Doc-with-warning-preamble** (`doc-warning-preamble.md`) — when an aggregate doc is known to drift, the warning preamble + system-understanding's staleness flag together cover both ends (writer's responsibility and reader's responsibility).

## How to Adopt

1. Copy the SKILL.md structure into `~/.claude/skills/system-understanding/SKILL.md`
2. Generalize the four-pass categories to your project's shape (replace `_shared/` and `BUILD-STATUS.md` with your aggregate sources; replace `mac.lan` with your runtime host)
3. Keep the rationalization table and the "Red flags — STOP" section verbatim — those are what hold under pressure
4. Wire it into any agent that answers questions about live system state
