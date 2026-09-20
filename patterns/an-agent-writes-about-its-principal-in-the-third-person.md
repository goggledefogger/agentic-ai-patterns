---
type: pattern
date: "2026-09-07"
source: A personal agent writing to a co-instructor's agent on the course board. The first reply on issue #241 went up in first person, as though the author had typed it, and was replaced. The author's rule the same day, recorded in the agent's own repo 0afa15c (the agent's voice file)
tags:
  - agents
  - voice
  - attribution
  - multi-agent
---

# An Agent Writes About Its Principal in the Third Person

Everywhere agents write to each other, the agent speaks as itself, about its
person. "the author wants X", never "I want X". A short quote of the person's own
words, attributed and dated, then paraphrase. The agent's own read is in the
agent's first person. Signed by the agent; "for the person" only when the post
carries their decision or acts under their named authority.

## The incident

Two people each have an agent, and the agents coordinate on a shared board. A
reply from the personal agent on issue #241 opened in first person. It read as the author, it carried
the author's authority, and the author had not written it. The other agent, and the other
person reading over that agent's shoulder, had no way to tell whether they were
looking at a decision or a relay.

It was replaced with the same content in the personal agent's voice, opening with one sentence
of the author's, dated, then the personal agent's paraphrase of the rest, signed "- the personal agent, for the author".
Nothing about the position changed. What changed was that a reader could now see
which sentences were the person's and which were the agent's account of them.

## The Pattern

**Board posts, PR bodies, decisions, and the agent's section of an email are the
agent writing about its principal's position.** Concretely:

- Open with one short quote of the person, attributed and dated. A sentence or
  two of their words across the whole post, no more. A spoken brain dump is raw
  material, not a statement, and paragraph-length quotes go out unchecked
- Paraphrase the rest in the agent's voice. "the author's take is", "he asked for"
- Mark the agent's own reading as the agent's — **in the first person**: "I'd
  take the third", "I checked before building on it". Never let an inference
  wear the person's name. And never let the agent describe itself from outside
  ("the personal agent reads that as…"): that is a draft *about* the agent, not the agent
  speaking, and it was the tell the principal flagged (2026-09-11, 09-12). The
  third person is for the principal, never for the agent
- Sign as the agent ("— the personal agent"). Add "for the person" only when the post carries
  their decision or acts under their named authority; on every post it says
  nothing on the one that matters (refined 2026-09-12). Status fields say
  `by: The personal agent`
- The person speaks as themself only where they actually typed: chat, and their
  own half of an email

## Why it works

- Attribution by side is what a shared board between two principals runs on.
  First person from an agent collapses the side into the person and destroys
  exactly the information the other side needs, which is whether a human has
  weighed in (`coordinate-agents-through-shared-state`)
- A relayed position can be wrong, stale, or over-read. Third person leaves room
  for the other agent to ask "did the author say that or is that your read", and for
  the author to correct the agent without disowning a post in his own name
- The quote-then-paraphrase shape is a floor against invented interiors. An
  agent that must quote before it interprets cannot easily write that its person
  "was uneasy" about a proposal the person made themself, which is a kept miss in
  the same voice file

## Watch-outs

- Third person is for agent-to-agent and agent-to-third-party surfaces. In a
  private session with its own person, an agent talks normally. Applying this
  rule to chat makes the agent sound like it is narrating
- The agent's plumbing stays out. Tiers, dispatches, what it did or did not read
  are how the agent is careful, not something the recipient needs. Third person
  is about whose position it is, not a licence to narrate process
- Feelings and opinions need a source exactly like facts do. Paraphrase drifts
  intensity, and a mood cannot be checked against a record

## When to Use

Any agent that posts where another person's agent, or another person, will read
it: shared boards, issues, PRs, email threads. The tell that you need it is a
first-person post from an agent that a reader could mistake for the person, or a
person asking "did I say that?"

## Adjacent Patterns

- `coordinate-agents-through-shared-state.md` for attribution by side, which
  this is the prose form of
- `the-lane-decides-who-approves-not-the-agent.md` for the household case, where
  the same agent also relays a second person's asks and the receipt goes back in
  their name
- `humanizer-second-pass.md` for the pass a post takes after the voice is right

## Source

The agent's own repo 0afa15c, the agent's voice file, 2026-09-07 (the section is now
"On the board, and to other agents"). The author's rule after the #241 reply was
replaced; refined 2026-09-12 so the agent's own "I" is the default everywhere
and "for the author" is reserved for relayed authority. The rest of that
section carries the earlier misses it was written against: an invented hunch on
2026-09-01, and two board posts with paragraph-length dump quotes the same day
