---
type: pattern
date: "2026-05-02"
source: Reddit u/Jonathan_Rivera, "How I use Obsidian as the long-term memory backbone for my AI assistant" (r/hermesagent, 2026-04-23, 704 pts)
tags:
  - memory
  - vault-architecture
  - context-engineering
---

# Three-Tier Memory Pipeline

Split assistant memory into three tiers with explicit promotion rules: hot memory injected every turn, vault living files read on-demand, daily notes as a session-log timeline. Promote between tiers on a numerical threshold, not vibes.

> **Companion to the main guide.** `claude-code-obsidian-guide.md` already discusses memory layering across CLAUDE.md, project files, Home.md, and daily notes — and lays down the *"daily notes are session logs, not an inbox"* philosophy. This pattern formalizes the layering as three named tiers with promotion rules. It does **not** override the session-log discipline: tasks still live in their destination files, the daily note is the timeline of what happened, not a task container.

## The Problem

The two failure modes for assistant memory are *bloat* and *amnesia*. Bloat: every correction, preference, and recent fix lands in the system prompt, the file balloons, the model stops reading it carefully, costs go up. Amnesia: the assistant has no persistent memory at all — every session re-explains the same things and re-makes the same mistakes.

The naive fixes both fail. "Just keep memory short" requires constant manual pruning and you lose useful context. "Just attach a giant knowledge base" reproduces the bloat problem one layer up.

What works is *tiered* memory with explicit promotion: most facts don't need to be in every turn, but they do need to be reachable.

## The Pattern

Three tiers, each with a defined access pattern and size budget:

### Tier 1 — Hot Memory (every-turn, ~6–9K characters)

Lives in a file the assistant loads at the start of every turn. Common shapes: `MEMORY.md`, `USER.md`, `CLAUDE.md`, `AGENTS.md`, `SOUL.md`. Contents:

- Active project state and ongoing corrections
- Communication preferences and procedural quirks
- Identity facts (name, timezone, family, key contacts)
- Recent feedback the user gave that hasn't been internalized yet

The size budget is the key. Pick a number (the original Reddit post uses 9K characters; ~6K is also workable depending on the harness's prompt budget) and treat it as a hard cap. When hot memory hits ~67% of the cap, promote stable entries to Tier 2.

### Tier 2 — Vault Living Files (on-demand, no fixed size)

Stable reference material the assistant reads when deeper context is needed. The original Reddit post groups these under `System/Assistant/` with three canonical files:

- `context.md` — operations, health, family overview
- `preferences.md` — communication style, delivery rules
- `environment.md` — hardware, services, known issues

In a Claude-Code-style vault, the equivalents are project `CLAUDE.md` files, the vault root `Home.md`, dedicated reference docs like `ROADMAP.md`. They aren't loaded into every turn; the assistant fetches them when the current task touches them. They can be hundreds of KB total without affecting per-turn cost.

### Tier 3 — Daily Notes (session-log timeline, append-only)

A dated note created each day at `Daily/YYYY-MM-DD.md`. Captures what happened: meetings discussed, decisions made, issues hit, things noticed. Append-only.

This is the tier where the main guide's *"session logs, not an inbox"* discipline lives. The daily note is **not** a task container or a schedule shadow — those belong in their destination files (project `CLAUDE.md`, calendar). The daily note records the conversation and the decisions; tasks that surface during that conversation get *routed* to where they live and *backlinked* from the daily note.

This tier is what gives the assistant time-travel: "what did I decide last Tuesday?" becomes a vault search, not a memory failure.

## The Promotion Rules

Promotion only works if it's mechanical, not judgment-soaked. Two rules:

**Hot → Living, at 67% capacity.** When `MEMORY.md` (or equivalent) hits ~67% of its size cap, scan for stable entries — facts that haven't changed in weeks, environment specifics, known failure patterns — and move them to the appropriate Living file. Leave hot memory at 30–50% headroom after the promotion. Don't run the scan more often than weekly; the churn isn't worth it.

**Living → Distillate, when noise dominates.** When a Living file gets large enough that the assistant has to skim past unrelated sections to find the relevant one, split it. The original keeps the high-altitude summary, the split-out file holds the detail. (See `living-doc-refresh-ritual.md` for the related stable / living / distillate shape.)

Don't promote in the other direction (Living → Hot) unless a stale fact is actively burning sessions. Promotion-back is a smell that the entry was wrongly classified.

## Content Routing (consistent with the main guide)

When the user says "log it" or "save it," the routing decides which tier the content lands in. This routing is the *same* routing the main guide describes — explicitly listed here so promotion thinking and routing thinking line up:

| Content type | Lands in |
|---|---|
| Operational events, decisions, things-that-happened | `Daily/YYYY-MM-DD.md` (Tier 3 session log) |
| Stable system facts, environment specifics | A vault Living file (Tier 2) — `environment.md`, project `CLAUDE.md` |
| Active corrections, communication preferences | Hot memory (Tier 1) — promote to Tier 2 when stable |
| Tasks and TODOs | Their destination file (project `CLAUDE.md`, `Home.md` TODO list) — *not* the daily note |
| Recurring workflows | A skill / procedure / template file — not memory |
| Unknown incoming | `Inbox/` until classified |

The routing rules belong in a Tier 2 file (often `preferences.md` or the project `CLAUDE.md`) so the assistant references them without burning hot memory budget.

## When to Use

- You're running an assistant with a persistent identity across sessions (Claude Code, Hermes, custom agents)
- You're seeing system-prompt bloat or repeated re-explanations of the same facts
- You have a vault or scratch space the assistant can read and write
- You want the assistant's memory to survive harness updates that overwrite system instructions

## When NOT to Use

- One-shot or short-lived assistants where there's no second session to remember things in
- Setups where the assistant has no persistent file storage
- Projects small enough that hot memory alone fits comfortably under cap

## Watch-outs

**The 67% number is a starting point, not a constant.** If your hot memory cap is small (a few KB), 67% may be too high — promote earlier. If it's large (20K+), 67% may be wasteful — let it run higher before scanning. Calibrate by how often the assistant references on-demand Living files vs. how often hot memory is actually re-read.

**Living files drift.** Without a refresh ritual, `environment.md` rots and starts contradicting reality. Pair this pattern with `living-doc-refresh-ritual.md` — verify against the running system, not memory.

**Don't break the session-log discipline by promoting tasks into the daily note.** The Reddit post's source template stuffs Tasks + Schedule into the daily note, which conflicts with this guide's main philosophy. Pick the routing in the table above instead — daily note is what *happened*, not what's *due*.

**Don't conflate "memory" with "context."** Hot memory is what's in the system prompt. Vault Living files are what the assistant chooses to read. Conflating them — putting on-demand reference into the system prompt "just in case" — undoes the whole point.

## Adjacent Patterns

- `morning-briefing-pipeline.md` — a way to surface Tier 2 / Tier 3 content as a daily push without violating the session-log discipline
- `living-doc-refresh-ritual.md` — the Tier 2 anti-drift mechanism
- `living-doc-archive-split.md` — the Tier 2 anti-bloat mechanism: when a Living file grows because closed items accumulate, split them to a non-auto-loaded archive
- `wire-into-existing-flows.md` — the routing rules need to be wired into a forcing function (CLAUDE.md, daily skill) or they'll sit unused
- `system-understanding-protocol.md` — citations from Tier 2/3 still need atomic-citation discipline; "the vault said so" isn't a source
