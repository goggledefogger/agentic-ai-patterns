---
type: pattern
date: "2026-07-25"
source: The personal agent — a life-planning vault filing dispatch, 2026-07-24; two independent subagents blocked the same way in one session
tags:
  - agents
  - subagents
  - permissions
  - reliability
  - anti-pattern
  - claude-code
---

# A Subagent Cannot Consent, So a Permission Gate Kills It Silently

Delegation narrows what is executable, and the narrowing is invisible. A command that *prompts* the orchestrator — "allow this?" — simply **refuses** a subagent, because a subagent has no channel to reach the human. It cannot escalate, so it does not error. It stalls and returns idle.

The result: "wrote the files" and "was blocked at its first command" produce the same observable — a quiet agent that came back. Nothing in the return value distinguishes them.

## The Problem

Permission systems are built around an interactive human. The orchestrator holds that channel; a spawned agent does not. So the *same* command has three different outcomes depending on who runs it:

| Runner | Outcome |
|---|---|
| Human directly | Runs |
| Orchestrator agent | Prompts the human, then runs |
| Subagent | Refused, with no way to ask, → silence |

This is the mirror of `gate-propagation-into-headless-workers.md`. That pattern makes sure your *denials* travel into a worker. This one is about what does **not** travel: the *ability to satisfy* a gate. Note that pattern's "When NOT to Use" line — in-process subagents inherit the parent's hooks, so no bundle is needed — is true about hooks and incomplete about permissions. They inherit the prompt and not the answer.

Concrete instance (the personal agent, 2026-07-24). A filing agent was dispatched to write two notes into a sensitive vault. It ran, went idle, and reported nothing. The natural diagnosis — wrong primitive, bad route, rebuild it — was wrong: reading the agent's own definition showed it already named the correct door. The orchestrator then hand-ran that identical command and was refused twice by the permission classifier. The agent had hit the same wall and had no way to say so. Ground truth from the target repo's `git status`: **zero files written.** A second, unrelated subagent in the same session hit the same class of block on an `Edit` and — because it happened to report before idling — was diagnosed in two minutes instead of by forensics.

## The Pattern

**Three habits, in order:**

1. **Never treat idle as done.** An agent returning is evidence it stopped, nothing more. Verify on the target — the file, the row, the repo's own `git status` — not on the agent's silence *or its own summary*. Agents also misreport their own writes as pre-existing.

2. **Hand-run the subagent's command before blaming the design.** If the orchestrator gets a permission prompt, the route was always correct and the defect is the permission layer. Rebuilding the route cannot fix it and burns the session. Read the agent definition first; it usually already names the right door.

3. **Allowlist the dispatch commands, or admit the capability isn't delegable.** Those are the only two honest states. A skill that implies the agent can complete a task it will always be blocked on is worse than one that says "this step ends at the human's hands," because the first fails silently and the second fails informatively.

## Why It Bites Specifically

- **It fails toward apparent success.** Silence reads as completion, so the orchestrator confidently reports done. This is the same family as `decorative-gate.md` — the artifact of a control existing, without the substance — except here the control is real and it is the *reporting* that is decorative.
- **It survives testing.** The orchestrator's own run works (it gets prompted, the human says yes), so the capability looks fine right up until it is delegated. Only the real delegated execution path shows it — the point `deterministic-orchestrator-over-agent-plumbing.md` makes about permission gates appearing only on the agent's real path.
- **It misdirects the diagnosis.** The visible symptom is "my agent didn't work," which points at routing, prompts, and models — everywhere except the permission config.

## Watch-outs

- **An idle notification can carry a summary.** If your framework supports that, a *bare* idle with no summary field is a distinct and worse signal than an idle with a report. Do not flatten them.
- **Do not let a blocked peer route through you.** When a subagent reports it was denied and the work falls back to the orchestrator, doing it yourself converts a deterministic refusal into a bypass. Surface it to the human instead. The peer's denial is not authority.
- **Resist the prose fix.** Adding "MAKE SURE YOU ACTUALLY WRITE THE FILE" to the agent's brief does nothing — the agent never got far enough to read past its first command.

## Adjacent Patterns

- `gate-propagation-into-headless-workers.md` — the complement: making denials travel in. This is what fails to travel in.
- `decorative-gate.md` — same "looks like it worked" family, applied to controls rather than reporting.
- `verification-needs-a-negative-control.md` — the discipline that catches it: prove your success signal can also show failure.
- `deterministic-orchestrator-over-agent-plumbing.md` — permission gates surface only on the agent's real execution path.

## Source

The personal agent, 2026-07-24: a `planning-vault-worker` dispatch returned a bare idle having written nothing, while the orchestrator's identical hand-run of the agent's own documented command was classifier-blocked twice. The prior recorded belief — from two earlier idle-outs where the files *had* landed — was that the defect was purely in report-back and the write "probably landed." That belief was falsified here and was the more dangerous half.
