---
type: pattern
date: "2026-07-20"
source: The course — a shared GitHub Projects board (status) as the task/roster source of truth for two instructors and their agents
tags:
  - architecture
  - source-of-truth
  - github-projects
  - automation
  - multi-agent
---

# External System as Source of Truth, Vault as Pointer

The exact inverse of `vault-as-cms-publisher.md`. There, the vault is the store and an external site is the derived view. Sometimes the canonical store belongs **outside** the vault — in a shared system (GitHub Issues/Projects, Linear, a DB) that several people and agents read and write — and the vault holds **pointers**, not the data.

## When to flip

A private vault cannot be the source of truth for state that is **shared across principals who each have their own vault** — the others can't see it. The moment two people (and their agents) need one answer to "what's the status of X?" or "who's in this cohort?", the truth has to live somewhere both sides reach. Flip to an external store when the state is (a) shared across separate vaults, (b) a live multi-writer surface, (c) already served by a system with an API and a UI you'd otherwise rebuild.

## The Pattern

- **The external system owns the shared STATE; the vault still owns CONTEXT.** The board owns task/roster *status*; the vault keeps planning, notes, per-person nuance. Name that line explicitly or you get the duplication the "one source of truth" rule forbids.
- **Markdown demotes to a pointer.** A note may reference a board item; it never re-tracks it. An action item written into a living doc is a reminder to *go look at the board*, and the board is truth. (See `memory-substrate-selection.md`: structured store + regenerable view.)
- **Deterministic code owns the writes to structured fields.** The model classifies (task vs person, which workstream); a script performs the write through the system's contract. Never hand-edit a *derived* field — see `deterministic-orchestrator-over-agent-plumbing.md`.
- **Map the automation boundary BEFORE designing automation.** For any external store, know what is scriptable (CLI/API) vs UI-only. GitHub Projects v2: item membership and field values are scriptable (GraphQL/`gh`); **views and workflows are UI-only** (no API at all). Designing a "set up the board" automation without this map wastes effort on the half that can't be scripted.

## Two traps that cost real time

- **"Commenting ≠ adding."** In GitHub Projects, an issue can be referenced, cross-linked, and commented on without ever being *added* as a board item — so it never appears on the board. The classic "the cards don't show up" bug is exactly this: reports were filed on the issues, but the issues were never added. Adding is an explicit, separate operation.
- **Forward-only automation never backfills.** GitHub's "auto-add to project" workflow only catches items created/updated *after* it's enabled. Always pair a forward-only rule with a one-time backfill script for the existing items.

## Why it works

One shared surface, no cross-vault syncing, and errors don't compound (any derived view rebuilds from the store). It also sets up a future custom front-end for free: the store is already an API, so a tailored UI is additive, not a migration.

## When to Use

- State must be shared across people/agents who keep separate private vaults.
- You need a live, multi-writer, human-in-the-loop surface (a board) more than a private notes file.

## When NOT to Use

- Single-owner state with no second reader — the vault is simpler and stays canonical (`vault-as-cms-publisher.md`).
- Below the scale where a shared system earns its keep (`memory-substrate-selection.md`'s "don't stand up a database for 150 notes" trap).

## Adjacent Patterns

- `vault-as-cms-publisher.md` — the inverse (vault is store, external is derived view)
- `memory-substrate-selection.md` — structured source of truth + regenerable compiled view
- `deterministic-orchestrator-over-agent-plumbing.md` — model classifies, a script writes
- `coordinate-agents-through-shared-state.md` — the multi-agent half: how two agents write the same store without colliding
- `dual-audience-views-over-one-store.md` — deriving human vs AI views over the one store
- `wire-into-existing-flows.md` — an always-on CLAUDE.md rule is what makes the routing actually fire

## Source

- The course `status` system: a shared GitHub Project (issues labeled `workstream`, append-only status comments, a generated `STATUS.md`, a derived board) used by both instructors' agents. The board is the humans' surface, `STATUS.md` the text one, the issues the store — three views, one source.
