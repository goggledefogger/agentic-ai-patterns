---
type: pattern
date: "2026-08-18"
source: The personal agent + the household agent, the property lane's first live sends — three failures in one evening, one after a full session reset
tags:
  - agents
  - prompting
  - reliability
---

# A Rule Must Live Where Matching Happens First

An agent with layered instructions resolves them in a fixed order, and that order is usually written down somewhere in its own context. A rule placed below the layer that already decided will never fire — no matter how correct, how emphatic, or how many times you rewrite it.

## The failure

A skill existed whose entire job was: when a property listing link arrives, hand it to another agent and do not evaluate it. Its description said so. Its body said so, with the negatives spelled out.

A listing link was sent three times. Every time the agent fetched the page and wrote a good summary. No job was ever opened.

The skill was deployed. The session was reset, ruling out stale context. The description was rewritten to lead with the trigger. Nothing changed.

The cause was one line in the agent's always-injected bootstrap:

> `## URL Pattern Rules (MANDATORY — check BEFORE routing table)`

URLs were matched against that block first. Listing links were not in it, so they fell past the skill into general behaviour every time. **The skill was competing in a race that had already finished.** Moving the rule into the mandatory block fixed it immediately.

## The pattern

Before writing a behavioural rule for an agent, find where the decision it needs to influence is actually made:

- **Read the agent's own precedence declaration.** Bootstrap files often state it outright ("check BEFORE routing table", "always run this FIRST", "MANDATORY"). That sentence is the map. Two "always first" rules in one context are a conflict to resolve, not an emphasis to add.
- **Rank by what is always in context.** A bootstrap-injected file beats a skill loaded on selection, which beats a file the agent must choose to open. Emphasis does not cross those tiers — a rule in a lower tier cannot out-shout a rule in a higher one, and adding capitals to it is the tell that you are debugging the wrong layer.
- **Fixing the same rule twice is the diagnostic.** If a rewrite changes nothing, stop rewriting. The rule is not weak; it is unreachable. The second edit is the signal to go looking for the layer above.
- **Confirm the layer, not the wording, with a reset.** A session reset that changes nothing rules out staleness and points squarely at precedence.

## The corollary that costs the most

Once the rule is in the winning layer, it is in *that agent's* winning layer only. Every other agent needs its own copy, each one drifts, and each re-learns the same lesson by failing in front of a person. That is the argument for a shared interception layer ahead of all of them — routing decided once, in front, rather than replicated into each agent's bootstrap.

## Relation to other patterns

`a-capability-contract-must-name-the-verb` says a contract must name what to DO, not only what to look at. This is the placement half of the same problem: a contract that names the verb perfectly, filed beneath the layer that already routed, is still never read. Naming the verb and placing the rule are both required; either alone fails silently, and silently is the expensive part.
