---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo)
tags:
  - documentation
  - decisions
  - adr
---

# Decision-Doc Pattern (Lightweight ADRs)

Dated architecture decision records at `docs/decisions/YYYY-MM-DD-topic.md` capturing what was decided, what alternatives were rejected, and what operational knowledge was learned.

## The Problem

Architectural decisions get made in commit messages, Slack threads, and Claude sessions. Six months later, nobody can find the reasoning. The decision gets re-litigated, or worse, reversed without realizing the original constraint still applies.

## The Pattern

A single directory `docs/decisions/` with one markdown file per decision:

```
docs/decisions/
  README.md                                    # template and filename convention
  2026-04-08-os-framing-and-agent-runtime.md
  2026-04-18-plow-vm-rebuild-and-imessage-live.md
  2026-04-18-humanizer-voice-second-pass.md
```

## Filename convention

`YYYY-MM-DD-short-kebab-topic.md`. The date is when the decision was made, not when the doc was written.

## Doc shape

```markdown
---
type: decision
date: "YYYY-MM-DD"
status: decided | superseded | under-review
tags:
  - decision
  - <topic-area>
---

# <one-line decision statement>

## Context
Why is this decision needed? What's the situation?

## Decision
What was decided, in one or two sentences.

## Alternatives considered
What else was on the table, and why each was rejected.

## Consequences
What this enables, what it constrains, what operational knowledge it bakes in.

## Review trigger
Under what conditions should this be revisited?
```

## When to Write One

- Non-trivial — you'll forget the reasoning in 3 months.
- Hard to reverse — changing it later is expensive.
- Externally visible — affects how the team works or how the system presents.

## When NOT to Write One

- Routine implementation choices (which sort algorithm, which lint rule). Commit message is fine.
- Decisions with no alternatives. If there was nothing else to consider, there's nothing to record.

## Example from the participant's system

`docs/decisions/2026-04-18-plow-vm-rebuild-and-imessage-live.md` captures:
- The VM rebuild playbook (nuke `rootfs.img`, relaunch Plow)
- That dashboard Communications → Channels raw-JSON editor is authoritative over direct `openclaw.json` edits (which get reverted by the guardrail sweep)
- That the dashboard "Running" indicator is not trustworthy — always cross-check VM logs

This is the kind of tacit operational knowledge that only comes from having been burned once. Without the decision doc, the next operator burns themselves the same way.

## Adjacent Patterns

- Pairs with the ROADMAP.md status-category pattern (`roadmap-status-categories.md`, template at `templates/roadmap.md`). The ROADMAP captures state, decision docs capture reasoning.
- Pairs with living-doc refresh ritual (`living-doc-refresh-ritual.md`). Living doc = what, decision doc = why.
