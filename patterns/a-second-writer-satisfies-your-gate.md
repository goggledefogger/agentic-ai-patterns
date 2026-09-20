---
type: pattern
date: "2026-08-03"
source: Walk-and-talk link-delivery-gate, ux-debt 61 — the author asked for the phone link twice while the gate sat wired with eleven passing tests
tags:
  - code
  - safety
  - anti-pattern
  - agents
---

# A Second Writer Satisfies Your Gate: Evidence in a Shared Store Names No Actor

A Stop hook existed to enforce one rule: while a walk bridge is starting up, the assistant may not end its turn until the phone link has appeared in an **assistant text block** of the session transcript. It was carefully built. It refused to accept the URL from tool output, because that UI collapses. It parsed rather than substring-matched, precisely so the token appearing in its own `curl` plumbing could not certify a delivery that never happened. Eleven tests, all green, including `test_url_only_in_tool_output_is_not_delivery`.

The walker still had to ask twice: *"is it not going to output the link here?"*, then *"you gave the qr code in this session but not the link."*

The gate was not wrong. It asked "does this URL appear in an assistant text block of this transcript?" and the answer was honestly **yes** — a *second process*, a tmux twin resumed onto the same session id, had written "here's your link" into that same transcript file minutes earlier. The human was reading a different surface and saw none of it. Two brains, one log; one brain's delivery discharged the other's obligation.

## The Pattern

A gate reads evidence from a store. The gate is sound only while the store has **exactly one writer** — the actor the gate is judging. Add a second writer and the evidence silently changes meaning:

- **Before:** "this line is in the log" ⇒ "I did it."
- **After:** "this line is in the log" ⇒ "*somebody* did it, somewhere."

Nothing in the gate breaks. No test fails, because the tests instantiate one writer. The predicate stays true and stops being *relevant*, which is the failure mode that survives review: a decorative gate has a passing branch reachable without the thing being true, and this has a passing branch reachable because **someone else** made it true.

Three questions before trusting any evidence store:

1. **Who can write here?** Enumerate processes, not roles. Session resumption, forked agents, twins, retries, and replays all multiply writers without changing any code you own.
2. **Does the record identify its author?** If the artifact cannot distinguish "I wrote this" from "a peer wrote this," the gate cannot enforce a per-actor duty — only a per-store one. Scope the check to what the actor itself emitted this turn, or stamp authorship at write time.
3. **Is the store the thing you actually care about?** Usually not. The requirement here was never "the URL is in the transcript"; it was "the human saw the URL." The transcript was a *proxy*, and proxies fail exactly where the thing and its shadow come apart — which for a shared log is at every additional writer.

## When It Shows Up

Multi-agent and session-resumption architectures are the natural habitat: any design where two processes intentionally share context (a twin that continues a conversation, a subagent appending to a parent's log, a retry replaying into the same stream) has by construction created a second writer for every shared artifact, including the ones being used as evidence.

The tell in review: a gate whose predicate is phrased over a *file* or a *store* rather than over an *action by a specific actor*. "The URL is in the transcript" versus "I said the URL." The first is checkable and wrong; the second is what was meant.

The honest fallback when the fix is structural and the hour is late: **record the reproduction, and demote the control back to an operator rule stated as load-bearing** — "say it yourself, every time, and never assume the gate is watching." A gate you know to be blind is far safer than one you still trust.

## Related

- `decorative-gate.md` — a control that returns "passed" without checking. This is one level up: the check is real, the *inference* from it is not.
- `half-gate-whole-verdict.md` — evidence for one axis, a verdict naming two. Same family: the verdict outruns what the evidence can distinguish.
- `announce-the-move-in-the-old-room.md` — the rule this gate was built to enforce, and which the twin's existence quietly re-broke.
