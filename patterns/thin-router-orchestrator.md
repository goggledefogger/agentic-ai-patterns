---
type: pattern
date: "2026-06-27"
source: Design session for "the personal agent," a personal chief-of-staff orchestrator over a constellation of Obsidian vaults plus Claude Code agents (research-report-driven, 2026-06-27). Cross-checked against Anthropic's orchestrator-worker Research system and the named household agent.
tags:
  - orchestration
  - architecture
  - claude-code
  - routing
  - subagents
---

# The Thin-Router Orchestrator

You want one agent that helps with everything across many vaults, repos, accounts, and other agents. The trap is building it as a brain that accumulates a model of your whole life. Build it as a **stateless router** instead. The human-facing name can be "chief of staff," but the architecture is a router that holds pointers and sensitivity tiers, never contents. Keep it empty.

## The Problem

A single front-door agent over a constellation of resources has one failure mode that dominates all others: it absorbs. Every delegated result, every preference, every contact, every credential it touches wants to live in its context. The "chief of staff" framing actively invites this, because a real chief of staff *knows things*.

The cost is not abstract. In Anthropic's multi-agent analysis, token usage alone explained roughly 80% of performance variance. The router's context grows with every result it pulls back in, so a thin router is the cost and reliability lever, not a matter of taste. A fat router is also the maximum security blast radius, because it ends up the one process holding sensitive data from every domain at once.

## The Pattern

**The router holds two small indexes and delegates everything else.**

- **Resource registry** answers *where and what*: one entry per resource with `{name, tier, where, how-to-reach, owner}`. Pointers and tiers only, no contents
- **Shared skills catalog** answers *how*: reusable `SKILL.md` playbooks any worker loads at the moment it needs them (see `shared-skill-symlinking.md`)

Workers are **subagents** by default (ephemeral, isolated context, return a summary, then vanish) and **named agents** only when the work genuinely needs a standing entity (see `delegation-decision.md`). Either way the worker does the work and the router stays small, because each delegation returns a conclusion, not the intermediate logs.

**The router is a git repo, runtime-agnostic.** Because it holds almost nothing, "run from anywhere" is just clone-and-point-a-runtime-at-it (Claude Code on a Mac or VM, a Hermes alien on a Pi, a thin messaging front-end later). The one machine-specific file is a per-host location map (logical resource name to local path). Keep that out of the router's brain so the same registry works on every host.

**Sensitivity tiers are both the routing key and the safety spine.** Tag every resource with a tier, for example open / personal-private / shared-IP, and give each a posture. Enforce the stops deterministically in code (`PreToolUse` deny or ask on tier paths and on every write or send tool, `PostToolUse` audit log, a kill switch), never as prompt instructions a model can talk its way past. Run defense-in-depth: a lightweight classify-and-pause at the router for the highest-sensitivity tiers, with the authoritative enforcement living downstream in each worker so it survives even if the router is bypassed. This is the orchestrator-side companion to `sensitivity-tiered-access-control.md`.

## Why It Works

- **Thin is cheap and reliable.** The router's context never balloons because results come back as summaries, so token cost and drift both stay bounded
- **Empty is safe.** A router with no contents and no broad credentials is a small blast radius even if a downstream agent is compromised by prompt injection
- **Two indexes keep concerns separate.** The registry can change (new vault, moved repo) without touching the skills catalog, and vice versa
- **Portability falls out for free.** A near-empty git repo moves trivially. The more the router holds, the harder it is to run anywhere

## The Naming Trap

"Chief of staff" tempts you to grant memory, preferences, and contact lists. "Router" reminds you to keep it empty. Document and build it as a router even if you call it something friendlier in conversation. Every time the router accumulates state, you reintroduce the cost, context-bloat, and security exposure the whole pattern exists to avoid.

## When to Use

- One front-door agent fronting several vaults, repos, accounts, and possibly other named agents
- You need the same agent to run across more than one machine
- The domains differ in sensitivity and some must be gated before any action

## When NOT to Use

- A single vault with a single agent. There is nothing to route, and a router is pure overhead
- A genuinely accumulation-shaped assistant whose entire value is a deep evolving model of one domain. That is a named downstream agent, not a router (see `delegation-decision.md`)

## Watch-outs

- **The registry is the one thing that can drift.** Pointers go stale when resources move or get renamed. Wire a staleness check into a ritual so the drift surfaces (`self-reporting-staleness-check.md`)
- **"Just this once" state is how routers get fat.** A cached preference here, a contact there. Push every such thing downstream the moment you notice it
- **Prompt-based stops are not enforcement.** If the only thing stopping a sensitive action is a sentence in `CLAUDE.md`, it is unenforced. Move it to a deterministic hook

## Adjacent Patterns

- `delegation-decision.md` — when a worker should be a subagent, a skill, or a full named agent
- `registry-based-monitoring.md` — the JSON-registry-plus-auto-discovery shape the resource registry borrows
- `sensitivity-tiered-access-control.md` — the downstream half of the tier enforcement
- `shared-skill-symlinking.md` — how the shared skills catalog is defined once and reached globally
- `three-tier-memory-pipeline.md` — orthogonal memory layering for the downstream agents that *do* accumulate
- `morning-briefing-pipeline.md` — a read-only proactive job a thin router can schedule once the gates are proven

## Source

- Anthropic, "How we built our multi-agent research system" (orchestrator-worker, lead agent delegates to isolated subagents, token use explains ~80% of performance variance)
- `~/src/household-agent` — the "alien" agent template (the household agent): per-agent `SOUL.md` identity, `AGENTS.md` routing table, scoped 1Password vault, `scope-policy.yaml` read-only scopes, deterministic startup gate
