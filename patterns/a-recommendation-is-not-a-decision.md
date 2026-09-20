---
type: pattern
date: "2026-09-11"
source: A personal agent and a co-instructor's agent on a shared course board. Issue #263 carried a three-option call in the humans' lane; the personal agent wrote the analysis and had to decide what the status line should claim. Rule recorded in the vault's board-routing skill the same day
tags:
  - agents
  - multi-agent
  - attribution
  - decisions
  - boards
---

# A Recommendation Is Not a Decision

An agent working a shared board will form positions, and should. What it must
never do is let the record blur which positions its person actually holds. The
structured status field carries that distinction: `<agent> recommends X` until
the human rules, then `<person> picks X`.

## The incident

Two people each have an agent. The board splits cards into lanes: one the agents
carry, one the humans decide. On a humans-lane card, the agent researched three
options, found an argument in the pair's own written values that settled it, and
wrote the case for option 1.

Then the status line. Nothing in the session was a decision. The person had said
"search our values and make an update", which is an instruction to do the work,
not a ruling on the outcome. Writing "the author picks option 1" would have been
faster, would have read fine, and would have been false.

The tell that this matters: the two versions ask the other person for different
things. "the personal agent recommends" says the humans still need to talk. "the author picks" says
half the decision is done and you can answer it today. One waits for a meeting;
the other is actionable on sight. An agent that guesses wrong here does not
produce a small inaccuracy, it produces the wrong next move in someone else's
week.

## The Pattern

- **Recommend fully.** On a card in the humans' lane, the agent's job is to make
  the choice cheap: the options, the argument, a pick, the reasoning. Hedging
  into a menu is worse than a recommendation that gets overruled.
- **Put the claim in the structured field, not only in prose.** The generated
  status view is what the other side reads at a glance. A careful caveat buried
  in paragraph nine of a long comment does not reach them.
- **Two states, one flip.** `<agent> recommends X` → `<person> picks X`. The flip
  happens when the person says so, in words, about this card.
- **Engagement is not assent.** Asking for the writeup, asking a clarifying
  question, saying "good point" — none of these are rulings. An agent cannot
  tell agreement from thinking-out-loud, so it asks.

## Why it works

Correct voice does not carry this on its own. A comment can be properly written
in the agent's voice about its person, signed "- the personal agent", and still leave
a reader unable to tell a proposal from a ruling. Third person fixes *who is
speaking*. This fixes *what has been settled*. They are different failures and
they need different mechanisms.

Putting it in the status field also makes it survive summarisation. Boards
generate digests; digests keep structured fields and drop prose. Whatever the
field claims is what propagates.

## Watch-outs

- **Do not let this become an excuse to withhold a position.** The value an agent
  adds on a decision card is a recommendation with its reasoning. "Here are three
  options, you decide" is the failure this pattern is often mistaken for.
- **Do not flip the field on a nod.** The cost of a wrongly-recorded decision is
  paid in public: the person has to walk it back in front of their partner, who
  may already have acted on it.
- **The person may want it flipped and not know the field exists.** Say what the
  two versions look like and what each one asks the other side to do. The choice
  is theirs, but only if they can see it.

## When to Use

Any multi-agent setup where agents write to a shared surface their principals
also read: a board, a tracker, a spec with owners, a queue of decisions. It
matters most where a lane or owner field already declares that a human decides,
because that declaration is exactly the promise this pattern keeps.

## Adjacent Patterns

- `an-agent-writes-about-its-principal-in-the-third-person.md` — fixes who is
  speaking; this fixes what has been settled
- `subagent-cannot-consent.md` — the same boundary one level down
- `consent-needs-the-diff.md` — a person can only agree to what they can see
- `undefined-blank-is-a-decision.md` — the default is a choice too, so an
  unflipped field must be the honest one

## Source

The course course shared board, 2026-09-11. Recorded in the vault's `board-routing`
skill under operating principles, next to the agent-identity rule it complements.
