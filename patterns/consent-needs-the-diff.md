---
type: pattern
date: "2026-08-05"
source: The personal agent's walk-and-talk bridge (server.py _read_prompt, SKILL.md invariant #9, spec-walk-silent-turn-incident)
tags:
  - safety
  - approvals
  - mobile
  - voice
---

# Consent Needs the Diff

An approval surface has a bandwidth: how much context the approver can actually absorb before answering. A desk session showing a full diff is high-bandwidth. A phone page mid-walk offering "1: Yes" to a spoken question is nearly zero. The decision being approved has a bandwidth requirement too — and the failure mode is a forwarder that matches them blindly.

## The Pattern

Classify every approval by the context informed consent requires, and route it only to surfaces that can carry that context:

- **Low-context approvals** (edit this draft, run this read, continue) — forward anywhere, including a tap or a spoken yes.
- **High-context approvals** (an agent's own gates: hooks, permission settings, policy files, path maps, routing rules, tier definitions) — desk-only, because consent without the diff is not consent. On a low-bandwidth surface these are **not deferred, they are converted**: auto-deny the prompt, draft the proposed change somewhere reviewable, tell the approver in one line what was parked and why. The walk drafts; the desk commits.

The classifier fails closed: anything that plausibly touches the agent's own safety machinery is high-context. A benign prompt held for the desk costs minutes; a safety prompt approved on a wrist costs the gate itself.

## Why Auto-Deny, Not Hold

An unanswered modal wedges the agent silently — the approver hears nothing and the session does nothing, which is a second incident on top of the first. Deny keeps the agent moving, the draft preserves the work, and the spoken line preserves trust.

## Where It Was Earned

2026-08-05: The personal agent's walk bridge forwarded every Claude Code permission dialog to the phone, unfiltered. A settings.json hooks edit — squarely the agent's own gate — was approved by voice mid-walk, violating the skill's own written invariant. The invariant existed; no code read it. A rule a forwarder doesn't consult is documentation, not a control.
