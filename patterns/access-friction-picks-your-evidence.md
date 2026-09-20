---
type: pattern
date: "2026-08-11"
source: The personal agent — researched a vendor's public docs to verify a claim about the author's own account while the stronger, account-specific answer sat in his gated vault, and reported a fact as "unverified" that he had verified himself six days earlier
tags:
  - agent-safety
  - verification
  - gates
  - research
  - anti-pattern
---

# Access Friction Picks Your Evidence

An agent with a gated local store and an open internet does not weigh its sources. It reaches for whichever one answers first, and the gate guarantees that is never the local one. The public web costs a tool call; the private record costs an approval, a dispatch, and possibly a refusal.

So the evidence hierarchy inverts silently. The **weaker, generic** source gets cited because it was frictionless, and the **stronger, specific** one goes unread because it was guarded. Nothing in the output shows this happened — the citation is real, the reasoning is sound, and the conclusion is merely less defensible than the one available all along.

## The incident

A support rep asserted something about a user's account. Verifying it, an agent researched the vendor's public documentation, found the rep contradicted by two vendor pages, and reported that as the finding. It also reported that a second claim — a per-account setting — "can't be checked from here."

Both conclusions were weaker than the truth, and both were fixed by one look at the user's own vault:

- The user had checked his **own console** six days earlier and written down what it displayed, with exact dates. That is account-specific evidence from the vendor's own product. It cannot be dismissed as "the docs describe a different product," which is precisely how a generic-documentation citation dies.
- The setting reported as uncheckable had been **confirmed on screen** by the user, and recorded, in the same note. The agent had told him something was unverifiable that he had personally verified.

The second failure is the worse one. "Unverified" is a claim about the world, and an agent that has not read the local record has no standing to make it. What it can honestly say is "I have not checked the place this would be recorded."

The public research was not wasted — two independent vendor pages made the case harder to wave off, and that mattered. But it was the supporting evidence promoted to lead because the lead was behind a gate.

## The Pattern

1. **Ask where this fact would be recorded before asking what is true.** If the question is about *this* account, *this* machine, *this* deployment, the local store is the primary source and the public doc is corroboration. Establish that ordering before the first search, because after the search you have an answer and no appetite for the harder path.

2. **A gate is a cost, not an absence.** "I can't reach it" and "it costs an approval" are different states, and only the second is usually true. Pay the cost or name it, and never let the cheap source quietly stand in.

3. **Never say "unverified" about a place you did not look.** Scope the claim to what you actually did: *not checked*, and where. This is the same discipline as `a-partial-read-proves-presence-not-absence`, applied to your own research rather than to a file.

4. **When both exist, lead with the specific and corroborate with the generic.** Account-specific evidence answers "is this true for me"; documentation answers "is this the rule." Presented in that order they reinforce; in the other order the specific one looks like an afterthought.

5. **Budget the local read into the plan, not the retrospective.** The cost is known in advance — one approval, one dispatch. It only feels expensive when it arrives after a conclusion already exists.

## Why it stays invisible

- **The cheap path produces a genuinely good answer.** Nothing is wrong with it; it is merely second-best, and second-best has no error signature.
- **Friction is felt, ranking is reasoned.** Reaching happens before deliberating, so the source is chosen before the question of which source is better is ever posed.
- **A gate teaches avoidance.** After one refusal, the guarded store starts reading as unavailable rather than as costly, and the agent stops proposing it at all.
- **The user rarely corrects it,** because the answer looked fine. This one surfaced only when the user asked for a second verification pass on work he had reason to care about.

## Watch-outs

- **This compounds with tiered stores.** The most sensitive material is the most gated and often the most authoritative — finances, credentials, health, contracts. The ranking inverts hardest exactly where accuracy matters most.
- **A summary of the local record is usually enough,** so the cost is smaller than it feels. Route the read through whatever lawful surface exists (`escape-hatch-is-the-denied-tool`, `lawful-write-surface`) rather than treating the whole store as off-limits.
- **Re-deriving what the local record already holds is the quiet tax.** In this incident the agent recomputed a reconciliation the user had already done and written down. It agreed, which felt like confirmation and was actually duplicated work.
- **Do not overcorrect into skipping the public source.** Two independent sources is the strong position; the pattern is about ordering, not substitution.

## When NOT to use

Genuinely general questions — how a protocol works, what a library does, what the standard practice is — have no local answer, and hunting for one in a private store is superstition. The pattern applies when the claim is about a specific system the user owns.

## Adjacent Patterns

- `a-partial-read-proves-presence-not-absence` — the same overclaim, scoped to a file rather than a research pass
- `escape-hatch-is-the-denied-tool` — why the local store was unreachable in the first place, and how to build the route
- `external-store-as-source-of-truth` — deciding which store is canonical *before* the question arrives
- `a-summary-can-invert-the-policy` — the failure mode of the cheap source, once you have chosen it
- `freshness-axis-must-match-the-question` — the other way a defensible-looking source answers the wrong question

## Source

The personal agent session 2026-08-11. Caught only because the user, about to send the conclusion to a third party, asked for everything to be double-checked — and then asked a second time whether his own knowledge bases had been consulted. They had not. Both corrections came out of the first look at them.
