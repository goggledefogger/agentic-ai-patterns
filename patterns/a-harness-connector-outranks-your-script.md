---
type: pattern
date: "2026-10-27"
source: A personal chief-of-staff agent running inside a dashboard app. The app's system prompt told every chat "your hosted connectors (mail, calendar, drive) are loaded; find them with tool search." Asked to verify an email the user had just sent, the agent went straight to the hosted mail connector, whose search index lags the mailbox, while the project's own live mail script was authorized and working. Over the 25 days after the host prompt grew, the project's own skills were invoked in 9.9 of every 100 sessions, down from 15.1, and context already loaded before the first reply rose from 54K to 90K tokens.
tags:
  - agents
  - routing
  - tools
  - context
  - multi-engine
---

# A Harness Connector Outranks Your Script Unless a Shared Rule Says Otherwise

When the host (a harness, an IDE, a dashboard, a hosted chat app) ships its own tool for a source your project already reaches with its own script, the host's tool wins by default. Its name and description sit in every turn's context. Your script waits behind a skill that must be invoked first. A host prompt that says "these connectors are available" reads to the model as "use these". Put a one-line routing rule where every engine reads it first: the host's tool is availability, not a route, and the project's skill for that source names the first route.

## The Problem

**Availability text works as a routing instruction.** "Your connectors are loaded, find them with tool search" is meant as a fact. At the moment of choosing a tool, it is the only routing text in view, so it decides. The project's rule ("personal mail goes through our script") lives in a skill the model never opened, because the host's tool was already one call away.

**The two routes do not reach the same things.** Hosted connectors commonly search a lagged index, can't fetch attachments, or see less than the project's direct API token. The connector returns something plausible, so the miss is silent: a just-sent email may or may not appear, and an absence reads like a fact about the mailbox.

**Host growth crowds out project skills.** As the host prompt and the listed skills grow, each project skill is a smaller share of what the model reads before choosing. Invocation of the project's skills drops even though none of them changed. The drop shows up only if someone counts it.

**A stale line in the project's own skill finishes the job.** If the skill still says "this source is connector-only" from before the script existed, even an agent that loads it is sent the wrong way.

**Hooks don't travel.** Fixing this with a pre-tool hook binds one engine. Codex, Gemini, IDE panes and other hosts run none of your hooks, and they get the same host connectors.

## The Pattern

1. **Write the rule once, engine-neutral, in the file every engine reads first** (the shared instruction file, or the playbook it tells every engine to read at start): "A connector the host offers is availability, not a route. For mail, calendar and files, the project's skill for that source names the first route; built-in tools come second."
2. **Make the first route a plain script**, not a host tool, so it runs the same under any engine and in headless runs.
3. **Keep the connector as the named fallback**, for when the script's token is dead. Don't delete it from the story; say when it applies.
4. **Fix the skill's own routing table and trigger words.** The trigger should match how people actually ask ("see the email I sent", "verify that email"), not only "scan my inbox".
5. **Count skill invocation over time.** Invocations per 100 sessions and context at first reply are cheap to compute from transcripts, and they are the only way to see crowding.

## Why It Works

The model picks from what is in view at the moment of choice. A rule in the always-read file is in view at that moment under every engine. A rule in an uninvoked skill or a Claude-only hook is not. Pointing at a script rather than a host tool makes the first route portable, and naming the fallback keeps the connector useful without letting it win by default.

## When to Use

- Your project has its own script or token for a source that a host also offers as a connector (mail, calendar, drive, issue trackers, chat)
- Your agent runs under more than one engine or host app
- A host's system prompt or skill list grew, and project skill use dropped with it

## Source

A chief-of-staff agent inside a dashboard app, 2026-10-27. A shorter version of the rule went into the agent's always-read playbook. The evidence went into the playbook's rationale file, to stay under the playbook's word cap. Related: [always-loaded-context-budget](always-loaded-context-budget.md), [reach-surface-resolution](reach-surface-resolution.md), [a-ruleset-travels-by-brief-and-by-hook](a-ruleset-travels-by-brief-and-by-hook.md).
