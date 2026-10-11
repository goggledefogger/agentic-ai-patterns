---
type: pattern
date: "2026-10-10"
source: The personal agent 2026-10-10 session, shipping a token-saving ruleset to the helpers it spawns
tags:
  - token-efficiency
  - skills
  - subagents
  - hooks
---

# A Ruleset Travels by Brief and by Hook: A Prose Skill Governs Only the Agent That Invoked It

A token-saving skill was loaded into the parent session. The parent obeyed it. The helpers the parent spawned did most of the reading, and they had none of the rules. The skill governed 1 agent and the work happened in several.

A skill is context for the agent that loaded it. It is not inherited. Every helper starts with a fresh context and only what the brief and the harness give it.

## The Pattern

1. **Put the rules in a block small enough to paste.** The shipped block is 80 words. The parent pastes it into every brief it writes. This door works in every harness, because it needs nothing but text
2. **Add a second door where the harness has one.** A start-of-subagent hook adds the same block to each helper automatically. It does not depend on the parent remembering
3. **Gate the hook on a visible activation.** The hook fires only when the session transcript shows the user invoked the skill: a user entry carrying the command tag, or starting with the command. It never fires because the plugin is merely installed. An always-on ruleset re-sent to every call costs more than it saves
4. **Do not count on the declarative route.** Preloading a skill into an agent definition was closed: a skill marked `disable-model-invocation: true` also cannot be preloaded
5. **Probe each door live.** A real helper was asked to quote the block back, once per door. Both quoted it
6. **Count the helpers on the card.** Report helper usage as the helper reported it. A missing number prints as `unavailable`, never as an estimate

## Why It Works

- **The brief is portable.** Text in a prompt survives any harness change
- **The hook is reliable.** It removes the human memory step on the harness that supports it
- **The gate keeps the cost honest.** The rules ride along only when someone asked for them

## When It Shows Up

Any ruleset, style guide or safety policy delivered as a skill, in a system that fans work out to subagents. The tell is a parent that behaves well while its summary quietly reports helpers that read whole files.

## Measured

On 2026-10-10 both doors were probed live on Claude Code 2.1.296. In each probe a real helper quoted the 80-word block back.

## Adjacent Patterns

- `token-discipline-useful-core.md` for the rules themselves
- `policy-on-every-door.md` for the same failure on entry paths instead of helpers
