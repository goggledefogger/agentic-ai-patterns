---
type: pattern
date: "2026-08-26"
source: The household-agent repo — deploy-to-pi.sh reset the agent's live session on every deploy while its own log line and CLAUDE.md both said it reset "when bootstrap/skills changed" (PR #274)
tags:
  - code
  - anti-pattern
  - debugging
  - operations
---

# An Unconditional Act Wearing a Conditional Name

A deploy script cleared the assistant's running conversation whenever it shipped new behaviour. Its log line said so:

```
Resetting active sessions (bootstrap/skills changed)...
```

The project's `CLAUDE.md` said so too, and went further, naming these deploys as *"the real mid-day context wipes"* that are *"unscheduled and frequent — that, not a timer, is what reads as random."*

It was not conditional. `_BEHAVIOR_CHANGED=true` was assigned at four sites that run on every deploy, and not one of them was guarded by anything. The skills phase set it three lines below its own message reading `51 skill(s) already identical on the Pi — not copied`. So every deploy destroyed the agent's context, whether or not anything about its behaviour had moved. The wipes read as random because they were uncorrelated with change, which is a stronger statement than "frequent" and points somewhere completely different.

## The Pattern

An action whose *name* carries a condition its *code* does not enforce is worse than an unnamed one, because the name is a standing explanation. Anyone who notices the symptom reads the label, finds a plausible cause, and stops. The documentation had already absorbed the behaviour and rationalised it: the qualifier "whenever bootstrap/skills changed" made an always-on wipe sound like an occasionally-annoying one, and that sentence is why nobody had looked in months.

The check is mechanical and takes a minute. For any side effect with a conditional-sounding name:

1. **Grep every assignment of the flag, not just the one you are reading.** Four sites, four different phases, one shared variable. Reading any single one in isolation looks fine
2. **At each site ask: what makes this line not run?** If the honest answer is "the phase didn't run" rather than "nothing changed", the condition is on *reaching the code*, not on the thing the name claims
3. **Distrust the documentation hardest when it explains the symptom well.** A doc that describes the behaviour accurately and attributes it to a plausible mechanism is indistinguishable from a doc that is right, and it retires the investigation either way

## Fixing half of it still produces the symptom

The first fix gated the two sites the log line actually named, skills and bootstrap. The deploy still reset. Two more sites, a TTS bridge deploy and a systemd unit-override copy, were also unconditional and neither is mentioned in the message or the docs. Worse, a written diagnosis produced an hour earlier had asserted those two "already fire only when their phase actually did something" — reasoned from the surrounding code, never measured, and wrong.

So the closing rule: **verify by identity, not by log text.** The proof that the fix worked was the gateway process ID being unchanged across a deploy, then moving when a single comment was added to one config file, then unchanged again after the revert. Log lines were the thing under suspicion. Asking them whether they were telling the truth would have been circular.

## When to Use

Any always-runs side effect with a conditional name: cache invalidation "on change", a restart "if config differs", a notification "when something breaks", a rebuild "when inputs move". The tell in review is a message or comment containing a condition, sitting next to an assignment that has no `if` above it. Reach for it especially when a behaviour is described as feeling random or intermittent by the people who live with it, because "uncorrelated with the thing we named" is the most common cause of that feeling and it is trivially falsifiable.

## Related

- `decorative-gate.md` — a control that cannot refuse. This is the mirror: an action that cannot decline to fire, described as though it decides
- `unrun-checks-read-as-passing.md` — the same gap between a claimed condition and an actual one, on the verification side
- `stale-pointer-asserts-confidently.md` — documentation that stays plausible after the thing it describes has changed

## Source

The household-agent repo, 2026-08-26. Traced from a session where the assistant's context was reset three times in one evening while nothing behavioural changed, found while running the live control for an unrelated deploy guard. Fixed by gating all four sites on bytes actually moving, with the copy left unconditional because a provenance marker and its TOCTOU re-check depend on it happening. The two `CLAUDE.md` sentences carrying the fictional qualifier were corrected in the same change.
