---
type: pattern
date: "2026-06-13"
source: A course participant's home-agent system (the workspace repo commit bfb5591 — runtime-rules.md Unattended-Run Discipline)
tags:
  - automation
  - agents
  - reliability
  - claude-code
---

# Unattended-Run Discipline

An agent running unattended (cron, launchd, scheduled GitHub Action) must fail loud and stop on trouble — never pause for input it can't get, and never improvise a self-repair. The helpful instincts that serve an interactive agent are hang risks when nobody's watching.

## The Problem

Code written for an interactive session assumes a human is there to approve a permission prompt, answer a clarifying question, or notice when the agent goes off-script. Move that same code to a scheduled context and those assumptions become silent failures: the agent blocks on a prompt that will never be answered, and the run hangs until it times out or is killed — usually discovered hours later when the expected output never showed up.

Two root causes, both observed in that system's scheduled routines (commit `bfb5591`):

1. **Compound commands that can't match a prefix allowlist.** A permission system that allows `git status` and `git log` by prefix will still prompt on `git status && git log` or a piped/`$()`-wrapped compound — the matcher can't prove the whole thing is safe. In an unattended run that prompt is a dead stop. (This is the same root cause behind the global ban on heredoc/compound git-commit invocations — compound shell defeats allowlists.)
2. **The self-repair detour.** This is the subtle one. A capable agent that hits a data error tries to *fix it* — and a Claude Code agent's environment is editable, so "fixing it" can mean reading and rewriting `.claude/settings.json` or other config. That system's `system-health-check` run on 2026-06-11 **froze trying to edit `settings.json`** after a data error. The agent wasn't broken; it was being helpful in a context where helpfulness is a trap.

The insight worth keeping: **in unattended mode, an agent trying to repair its own environment is a failure mode, not a recovery.** The right move on trouble is log-and-end, not improvise-and-continue.

## The Pattern

Add an explicit "Unattended-Run Discipline" section to the runtime rules / CLAUDE.md that scheduled skills read, and have each scheduled skill reference it. The core rules:

- **Never touch settings or config to recover.** No editing `.claude/settings.json`, permission files, or environment. If a tool call is blocked, that's a finding to log — not a problem to fix mid-run.
- **Log and end on data error.** When inputs are missing or malformed, write a clear diagnostic to the run log and exit cleanly. Don't retry-loop, don't escalate privileges, don't detour into repair.
- **No compound shell in scheduled paths.** One command per call, each matchable against the allowlist. No `&&`, no pipes into mutating commands, no `$(...)` wrapping the real work. (Pairs with the interactive-session rule against heredoc commits.)
- **No interactive prompts.** Anything that would ask a human (a confirmation, a disambiguation) must have a non-interactive default or be skipped-and-logged. Assume nobody will answer.
- **Make the failure visible.** A clean exit with a logged diagnostic is the goal — but the *absence* of expected output should itself be detectable (see the stall-detection half of `churn-free-generated-artifacts.md`: a task that stops running should flip a status cell, not fail silently).

## Anti-pattern: porting an interactive skill to a schedule unchanged

The most common way to hit this: a skill works great when you run it by hand, so you add a launchd plist and walk away. It hangs the first time it hits a prompt you used to answer reflexively. Before scheduling any skill, audit it for the four triggers above (compound commands, config edits, retry loops, interactive prompts) — the interactive version almost always has at least one.

## Anti-pattern: treating a blocked tool call as a bug to engineer around

When a permission prompt blocks an unattended run, the instinct is to make the agent smarter about getting past it. That's backwards — the block is information. Log *what* was blocked and end; then fix the allowlist (or the compound command) deliberately, in an interactive session, where you can reason about whether the thing should actually be allowed.

## How to Adopt

1. Write an `Unattended-Run Discipline` block in the runtime rules / CLAUDE.md that scheduled flows load.
2. In each scheduled skill, add a one-line reference to it (wire it — `wire-into-existing-flows.md`).
3. Audit every skill *before* you schedule it: compound shell? config edits on error? retry loops? interactive prompts? Fix all four.
4. Pair with stall detection so a clean-exit-on-error is still *visible* downstream — a silent success and a silent skip should not look identical.

## Adjacent Patterns

- **Churn-free generated artifacts** (`churn-free-generated-artifacts.md`) — its stall-detection half is what makes "log and end" visible: a stopped task flips a status cell instead of vanishing.
- **Wire adoptions into existing flows** (`wire-into-existing-flows.md`) — the discipline block only works if scheduled skills actually reference it.
- **Filesystem queue** (`filesystem-queue.md`) — durable queues are how an unattended run defers work it can't do now instead of blocking on it.
- **Detached launch from a time-boxed agent** (`detached-launch-from-timeboxed-agent.md`) — the interactive sibling: same "compound shell defeats allowlists" root cause, but with a human waiting the answer is detach-and-notify rather than log-and-stop.
