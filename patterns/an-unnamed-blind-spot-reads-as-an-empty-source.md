---
type: pattern
date: "2026-08-17"
source: First live test of the personal agent's Telegram walk lane (the voice route, takopi bridge on the cloud VM). Asked "what's on my plate," the bridge-spawned session answered from what it could reach and said nothing about the author's email — it had no lane to either mailbox from that host. Eyes-free on a walk, the omission read as "nothing in email." The reach gap was real (personal Gmail had no headless lane anywhere); the failure was that nothing declared it.
tags:
  - reporting
  - routing
  - voice
  - anti-pattern
  - multi-agent
---

# An Unnamed Blind Spot Reads As An Empty Source

A report that draws on several sources — a brief, a status gather, an agent answering "what's on my plate" — is read as a claim about *all* of them. The reader cannot tell a source that returned nothing from a source that was never consulted, unless the report says which. So every surface that answers on behalf of multiple sources must name the ones it could not see, in one clause, every time. Silence about a dead or unreachable source is not neutral: it inherits the meaning "that source is empty," which is a confident false answer nobody typed.

## The Problem

The same brain runs in more than one place, and reach is a property of the *host*, not the conversation. A desk session reads personal mail through an interactive connector; the same assistant reached over Telegram runs on a headless box where that connector does not exist. The conversation feels continuous to the user, so the user's model of "what the personal agent can see" carries over — and the first time the remote host is asked about a source only the desk can reach, the honest answer is "I can't see that from here."

The failure mode is that no one wrote that sentence down anywhere the remote brain loads. The session answered the parts it could, omitted the part it couldn't, and the omission was invisible *as an omission*. On a screen, a reader might notice the missing section. Eyes-free — a voice reply on a walk — there is no layout to notice a hole in; whatever is not spoken does not exist.

This is the surface-level sibling of two existing patterns. `unknown-value-renders-as-absence` is the record-level version: a value outside the vocabulary silently inherits the meaning of the blank. `reach-surface-resolution` governs *resolving* reach — a resource has several backing stores and several doors. This pattern is about *reporting* reach: once resolution says a door is closed from here, that fact must travel to the consumer, because the consumer's default interpretation of silence is wrong.

## The Pattern

1. **Declare reach where the agent loads it.** Each host or worker profile (its CLAUDE.md, route profile, or system prompt) states what that brain can and cannot see from there — not as a caution in a doc humans read, but in the instruction file the answering session actually consumes. Reach declared anywhere else is reach the answer will not reflect.
2. **A source you cannot see gets one clause, never silence — and never a paragraph.** "Your personal inbox I can't see from here" and move on. The clause is the entire fix for the *report*; a lecture about OAuth is a different failure (the plumbing leak), and silence is this one.
2b. **Naming the wall is half the job — the profile must also carry the routing verb.** Hours after the disclosure fix landed, its sequel: asked for work that belonged to the unreachable store, the agent said the perfect clause and then let the task evaporate — nothing parked, nothing handed to the host that could do it. The same agent family had already invented the right move that morning (park a task file in a cross-host handoff queue, open a PR, say "parked for the desk"), but the invention lived in one session's context and died with it. So the declaration in rule 1 is a triple, not a pair: the host's own reach, the global map, and *where out-of-reach work goes*. Two of three still drops tasks, and on an eyes-free surface nobody sees them fall.
3. **Distinguish "empty" from "unseen" in the report's own vocabulary.** A gather that could not run a source must not let that source's section render as zero items. Zero is a claim; unreachable is a disclosure.
4. **When reach changes, the declaration changes in the same commit.** A wired lane whose profile still says "can't" produces the inverse failure: the agent refuses or disclaims a thing it can now do. The declaration is load-bearing config, not documentation.

## Why It Works

The consumer's prior does the damage. Nobody assumes a report is partial unless told; a partial report silently masquerades as a complete one, and the more fluent the report, the stronger the masquerade. Naming the blind spot converts an invisible gap into a visible boundary — and a visible boundary is actionable (wire the lane, ask at the desk) where an invisible one just produces wrong beliefs.

The one-clause discipline matters as much as the disclosure itself. If declaring a gap costs a paragraph, the answering session will learn to skip it; if it costs six words, it survives every register, including a spoken one-breath reply.

## Trade-offs and Limits

- **Scope the disclosure to sources the question implicates.** "What's on my plate" implicates mail and calendar; it does not require reciting every unreachable store the host knows about. A blanket recital is noise wearing honesty's clothes.
- **The declaration is a cache of a measured fact.** It rots like any pointer (`stale-pointer-asserts-confidently`): verify a "can't" before repeating it forever — the 2026-08-17 case closed its gap the same day, and the profile had to flip from "no" to "yes, with a weekly expiry" in the same sitting.
- **This does not replace liveness checking.** A declared-reachable lane that is currently dead needs the monitoring pattern (`known-broken-must-not-page` and its alarm); the declaration says what *should* work from here, the checker says what does today.
