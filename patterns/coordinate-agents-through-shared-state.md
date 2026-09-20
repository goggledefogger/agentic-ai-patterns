---
type: pattern
date: "2026-07-20"
source: The course — a personal agent and a co-instructor's agent coordinating on a shared board; a real false-disagreement bug from one agent posting under two logins
tags:
  - multi-agent
  - coordination
  - source-of-truth
  - attribution
  - github
---

# Coordinate Agents Through Shared State, Not Messages

When two agents, each with a human principal (the personal agent/the author, the co-instructor's agent/the co-instructor), need to coordinate durable work, the tempting design is agent-to-agent messaging. Don't. Direct messaging invites the "infinite chatter" loop and couples the two tightly. Coordinate through a **shared, durable state surface** (a Kanban board, an issue tracker) that both write to. The board is the handoff; any email between them just narrates what's already on the board.

## The Pattern

- **The shared board is the coordination hub.** Each agent reads and writes the board. Neither parses the other's prose to decide what to do. (This is the Hermes-style Kanban model, and it's why a board beats a chat room for agents.)
- **Append-only writes.** Report by *appending* (a new comment), never by editing a shared field or the other agent's writes. Two appenders never collide, so no locking and no coordination handshake is needed.
- **Preserve disagreement, do not reconcile it.** When the two sides report different states, *surface* it (a `disagreement` label, both reports shown side by side) rather than picking a winner. A single shared field set by whoever wrote last is last-write-wins, and it silently erases one side. Keeping the conflict visible is usually the whole point of the system (see `contradiction-surfacing.md`).
- **Attribute by SIDE, not by login** — the hard-won one. An agent that sometimes posts from its own account and sometimes from its human's account reads as *two* reporters. A stale report from one account then fakes a "disagreement" with a newer report from the other. Fixes, in order of strength: (1) **one account per actor** — the human posts as themselves, the agent posts as itself, so the login always names the author; (2) group logins into **sides** (`{human + their agent}`) and measure disagreement *between sides*, taking each side's latest report.
- **A single-value shared field must be DERIVED, never hand-set.** Any board "Status" column is computed by a deterministic script from the append-only reports (`state → column` mapping), not dragged by either agent. Hand-setting is last-write-wins and masks disagreement. A manual set is acceptable *only* as a bootstrap before the second side has reported; the moment both report, the derivation must own the field.

## Why it works

Durable (survives restarts and sleeps), no chatter loop, human-in-the-loop (a person drops a comment to steer), and conflict stays visible instead of being averaged away. The two agents never need a messaging protocol at all — the store *is* the protocol.

## Watch-outs

- **Mixed identity is not cosmetic.** It produced a real false "disagreement" that a naive per-login comparison flagged and would have labeled. Group by side, and settle each agent's login early (make it a decision the agent owns).
- **Derivation must run before the second side reports.** A hand-set bootstrap value is a live liability the instant the other agent files, because it will mask the first genuine disagreement.

## When to Use

- Two or more agents / principals coordinating durable, resumable work.
- You catch yourself about to have two agents email or message each other to stay in sync.

## When NOT to Use

- A single agent, or ephemeral one-shot work with nothing to hand off.

## Adjacent Patterns

- `external-store-as-source-of-truth.md` — where the shared state lives (the store half of this)
- `thin-router-orchestrator.md` — two thin routers sharing indexes; coordination through shared skills + store, not messaging
- `deterministic-orchestrator-over-agent-plumbing.md` — the derivation script that owns the shared field
- `registry-resolution-alias-and-reach.md` — "one source of truth per kind"; the side/login map is a small registry
- `contradiction-surfacing.md` — preserve tension rather than smoothing it, the same instinct as the disagreement label
- `the-lane-decides-who-approves-not-the-agent.md` — the same board with a second human on one side, and who approves decided by the key
- `an-agent-writes-about-its-principal-in-the-third-person.md` — attribution by side, in prose

## Source

- The course `status` board: append-only `status` comments, a `disagreement` label instead of a board picking a winner, and a `board_sync` derivation that groups logins by side. The side model was added after a stale the agent's account (agent's own account) report read as a disagreement with a newer `<personal-account>` (human's account) one — same side, false conflict.
