---
type: pattern
date: "2026-06-27"
source: Design session for "the personal agent," a personal chief-of-staff orchestrator (2026-06-27). The decision framework the personal agent applies whenever the user asks "should I build or automate this." Validated against the named household agent and the project's existing subagent and skill patterns.
tags:
  - delegation
  - subagents
  - skills
  - architecture
  - claude-code
---

# Delegation Decision — Skill vs Subagent vs Named Agent vs Direct

When you want to build or automate something, four paths look interchangeable: write a skill, spin a subagent, create a new named agent, or just use Claude Code directly. They are not interchangeable, and the expensive mistake is creating a named agent when a subagent and a skill would do. This is the framework for picking the lightest path that works.

## The Three Units (and the fourth path)

The confusion comes from collapsing three different things into the word "worker."

| Unit | What it is | Named? | Cost | Holds memory? |
|---|---|---|---|---|
| **Skill** | A playbook, reusable how-to (`SKILL.md`) | No (functional) | Cheap | No |
| **Subagent** | An ephemeral worker, spun up for one task in isolated context, returns a summary, vanishes | No, or functional like `calendar-reader` | Cheap | No |
| **Named agent** | A standing entity with its own identity, runtime, channel, memory, credentials | Yes | Expensive | Yes |

The fourth path is no new unit at all: **just use Claude Code directly in the relevant repo.** That repo's own `CLAUDE.md` is already the project-specific agent.

**Skills are not workers, they are what workers read.** A subagent or a named agent does the work and loads the right skill to do it well. Treating a skill catalog as the set of "delegated workers" is the common wrong turn, because it leads to inventing entities to "own" skills that need no owner.

## The Named-Agent Test

Create a *named* agent only when **two or more** of these hold:

1. It needs **standing memory** that accumulates over time (a home agent that knows your media library, the house, the pets)
2. It needs its **own runtime or availability** (always-on, a different machine)
3. It needs its **own channel or identity** you converse with directly (a Telegram presence)
4. It needs **broader or different credentials** than the orchestrator should ever hold (its own secrets vault, its own external account)
5. It runs **autonomously on a schedule** doing its own thing

Fewer than two: do not name an agent. Use a subagent plus a shared skill, or do it directly. No name, no vault, no ongoing upkeep.

## The Build Paths

| You want to... | Best path | New named agent? |
|---|---|---|
| Build or fix inside one existing project | Claude Code directly in that repo (the orchestrator can route you there with a spec) | No |
| A build spanning several resources | Orchestrate: subagents or headless `claude -p` per repo, tier-gated | No |
| Standing engineering judgment or standards across all projects | A shared skills layer the orchestrator points to | No, that is skills, not an entity |
| A standing, always-on, separately-credentialed presence others also use | A named agent, only if the test passes | Maybe, rarely |

## Worked Example — the "CTO agent" that should not exist

A tempting idea is a named "CTO" or "software-builder" agent. Run the test: engineering memory lives in each project's repo, not in a roaming builder (fails 1), it has no own runtime or channel (fails 2 and 3), credentials are already scoped per-project through the repo's own GitHub setup (fails 4), it does not run on a schedule (fails 5). Zero of five. A CTO agent is a costume on top of Claude Code that adds maintenance and buys nothing. Building software is Claude Code directly in the project repo, and the "CTO" you actually want is your shared skills layer (planning discipline, review gates, issue-splitting taste), loaded into whatever repo you open.

## Why It Works

- **It prices in the real cost.** A named agent means a vault, an identity, a deploy path, a scoped secrets store, and forever-maintenance. The test forces that cost to be earned, not assumed
- **It is the ponytail ladder applied to org design.** Default to the lightest unit that works, climb only when it genuinely does not
- **It removes the recurring anxiety.** "Do I need a new agent for this" becomes a checklist, not a vibe

## Make It a Standing Capability

The highest-leverage move is to encode this framework as a skill the orchestrator loads whenever you ask "should I build, make an agent, or automate X." The detailed analysis happens inside the skill, the output is one short recommendation. That turns a one-time decision into a repeatable separation-of-concerns advisor, which is one of the strongest reasons to have an orchestrator at all.

## When to Use

- You are about to "make an agent for that" and have not checked whether a subagent plus a skill would do
- You are deciding where a new automation should live
- You want the orchestrator itself to coach these choices rather than answer them ad hoc

## When NOT to Use

- The unit is already forced by a hard constraint (the work must run always-on on another machine, so it is a named agent by constraint). Skip the framework, the answer is given

## Watch-outs

- **The borderline is "it might grow into its own thing someday."** Someday is not two-of-five today. Build the subagent now, promote to a named agent when the test actually passes
- **A skill catalog is not an org chart.** Adding skills does not require adding agents. Keep the two indexes separate (see `thin-router-orchestrator.md`)

## Adjacent Patterns

- `thin-router-orchestrator.md` — the orchestrator that applies this framework and keeps the resource registry and skills catalog separate
- `subagent-ceremony-by-task-type.md` — once you have chosen subagents, how much review ceremony each task earns
- `shared-skill-symlinking.md` — how the shared skills layer is defined once and reached globally
- `quick-win-scaffolding.md` — when standing up a new system, scaffold the minimum and name the rest as seeds rather than building agents up front

## Source

- Anthropic, "How we built our multi-agent research system" (orchestrator delegates to isolated subagents that return conclusions)
- `~/src/household-agent` (the household agent) — a named agent that passes the test on all five axes: standing memory, own Pi runtime, own Telegram channel, own 1Password vault, scheduled heartbeats
