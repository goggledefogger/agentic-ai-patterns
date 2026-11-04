---
type: pattern
date: "2026-11-03"
source: The personal agent and its dashboard (2026-11-03): a request for a way to answer the password manager's biometric prompts from a phone turned into a count of the prompts by cause, and most of them guarded nothing
tags:
  - approvals
  - secrets
  - friction
  - gates
---

# An Approval That Guards Nothing Is Only Friction

The ask was a way to answer the password manager's fingerprint prompts from a phone, because they kept stalling work whenever the owner was away from the desk. The obvious build is a better channel for answering approvals. The better first move was to count them.

Across about 7 weeks of session transcripts there were roughly 300 prompting commands. Nearly all were the agent reading a secret through the human's desktop session, from a vault its own scoped, read-only service account already read silently. The fingerprint was protecting nothing. Anyone, or any process, that could ask the agent could get the same secret with no prompt by taking the other door.

It was worse than a useless prompt. The credentials skill told the agent to fall back to the human's desktop session whenever the service account hit its daily quota. That turned an occasional rate limit into a standing habit of prompting.

## The Pattern

Before building a better way to answer approvals, inventory them by cause and ask of each one what it protects that is not already open another way.

1. **Count by cause, not by total.** A pile of prompts feels like one problem. Grouped by the path that raised them, most turn out to be one redundant route
2. **For each cause, name what the prompt guards.** If the same secret is readable without the prompt through another identity, the prompt guards nothing. Route the work through the identity that already has access
3. **Delete the fallback that re-creates the prompt.** A skill or script that says "if the quiet path fails, use the human's session" will quietly bring the prompt back. Fail with a named error instead
4. **Keep the prompt where the approval is the point.** A deploy key, an encrypted backup disk, a work vault: these were set to per-use approval on purpose. They are the few left, and they are the ones worth a better channel, such as a per-action fingerprint step-up from any device

## Why It Works

- **A prompt is a cost paid per use.** If it buys no protection, the whole cost is waste, and people respond to waste by approving without looking, which spoils the prompts that do matter
- **The deliberate few become visible.** With the redundant ones gone, every remaining prompt is one somebody chose, and a faster channel for exactly those has a clear job
- **It targets the cause.** A phone approval screen would have made 300 pointless taps easier. Removing them made them vanish

## When to Use

When approvals are stalling work and the instinct is to make them easier to answer. Also when an agent has two identities that reach the same store, one quiet and one interactive. Skip it where the interactive identity is the only one with access, since there the prompt is the gate.

## Source

The personal agent, 2026-11-03. See `decorative-gate` (a control that returns "passed" without checking, the inverse of one that checks nothing yet still stops you), `access-friction-picks-your-evidence` (what friction does to an agent's choices), and `an-unattended-path-must-not-share-fate-with-an-interactive-one` (the same two-door shape seen from the unattended side).
