---
type: pattern
date: "2026-08-31"
source: The dashboard permission cards (the course repo #208) — four hook asks answered silently by a grant from an unrelated click; the personal agent #115 for the cleanup
tags:
  - agent-safety
  - approval-gates
  - consent
  - tool-design
---

# A Standing Grant Forgets Its Question

A "don't ask again" button captures one consent and stores one fact. If the stored fact is coarser than the question — the tool's name instead of the ask itself — every future question that shares the name is answered by a click that never saw it.

## The incident

A dashboard shows permission cards and offers "don't ask again" on every one. When the CLI attaches a scoped rule suggestion, the grant lands in the member's own settings, visible and revocable. When it does not, the server keeps the grant itself — in a `Set` keyed by **bare tool name**, consulted before any card is shown:

```js
// already answered once, for good — never ask a second time
if (alwaysAllow.has(req.tool_name)) { …respond allow, no card… }
```

The member clicked "always" on some benign `Write` card. Weeks of design later, a security hook started returning `ask` for writes into a protected vault, each verdict carrying its reason — *"one card per write, stop-first as ask-first."* Four asks in one session — two restricted reads, a write, a shell command — every one answered `allow` by the server within a second, no card, member watching the tab. The audit log showed the hook working perfectly; the page showed nothing at all. The entire safety column of a reviewed, merged, 131-tests-green permission design was inert, and the observation that "reads work at full fidelity" had been quietly re-explained: it is exactly what a swallowed ask looks like.

## Why the coarse key is attractive

The suggestion-less card is precisely the one that repeats most, so the pressure to make "always" work there is real, and the tool name is the only stable field in hand. The comment on the grant even said the right-sounding thing — *"already answered once, for good"* — which is true of the question that was asked and false of every different question that shares the tool name. **"Never ask a second time" quietly became "never ask a second question."**

Two aggravations compound it:

- **The grant is invisible.** It lives in process memory, appears on no surface, and dies only with the server — while its own comment claims a longer lifetime ("the life of the brain"). Nothing a member can read says the gate is off.
- **An asker cannot be told apart.** By the time the request reaches the answering layer, a security hook's `ask` (with a stated reason) and a routine mode escalation look identical. The one ask that must never be silenced is indistinguishable from the one that begged to be.

## The Pattern

1. **A durable grant carries the scope of the question it answered.** Tool name plus the stable part of what was asked — a path prefix for file tools, a command prefix for a shell — or no durable grant at all. A grant that cannot name its scope is a policy change, and policy changes do not ride in on a convenience button.
2. **A gate's ask is not silenceable by a grant the gate never saw.** If provenance survives to the answering layer, exempt hook-originated asks from standing grants; if it does not survive, that is the upstream bug to file first.
3. **Standing grants are a surface, not a variable.** Each one visible, each with a take-back, each with its honest lifetime. A grant nobody can see is a gate nobody knows is open.
4. **Verify a gate by watching it fire, not by watching work succeed.** "The protected operations completed" is what success looks like *and* what a swallowed ask looks like. The proof is the card appearing — the audit-to-execution gap (verdict at :49, execution at :50) was the only visible symptom here.

## Watch-outs

- The fix is not "remove the button." The repeat-ask problem it solves is real; the fix is a key as fine as the question.
- Process-lifetime grants reset on restart, which makes the failure intermittent across days — the gate works after every reboot until the first "always" click, so spot checks pass.
