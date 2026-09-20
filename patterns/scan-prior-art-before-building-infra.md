---
type: pattern
date: "2026-07-11"
source: The personal agent session (2026-07-11), staged a key-in-env-var commit-signing setup, then a two-minute scan surfaced a strictly better platform-signing approach that needed no key at all
tags:
  - workflow
  - planning
  - decision
---

# Scan for Prior Art Before Building Infrastructure

Before you build or stage an infrastructure capability, spend a couple of minutes scanning for an established way to get it. Infrastructure needs like signing, auth, secrets, deploy, and retries are almost always already solved, and the found approach is often strictly better than the first one you would have hand-built.

## The Problem

When a task needs an infra capability, the reflex is to build the first approach that works. You design it, test it, stage it, and only then discover that a well-known pattern would have given you the same capability with less code, less risk, and a better long-term shape. By that point you have sunk real effort into the worse option, and the sunk cost makes you want to ship it anyway.

The tell is that infra needs are rarely novel. Signing commits, holding a secret, authenticating to a service, deploying a static site, these have been solved many times. Building your own is usually reinventing, not inventing.

## The Pattern

Insert a short prior-art scan between deciding you need capability Y and building the thing that provides it.

- Name the capability in general terms, not your specific build ("get Verified commits from an ephemeral agent," not "write a script that materializes an SSH key")
- Do a quick external scan, docs and a couple of searches, for how this is normally done
- Weigh what you find against your constraints. If it is strictly better and cheaper, switch to it before you build. If your case is genuinely the uncovered one, build, but now you know why

Timebox it. This is a scan, not a research project. The point is to catch the strictly-better option before you commit code to the worse one.

## Why It Works

- Infra is well-trodden, so the odds that a cleaner established pattern exists are high, and the scan is minutes against a build of hours
- Catching it before you stage avoids the sunk-cost pull that makes a team ship the worse option just because it is already written
- The worse option is not only more work now, it is usually a worse shape to maintain later (a long-lived secret you must guard, a bespoke flow no one else understands)

## When to Use

- Any "I will build X to get capability Y" where Y is a common infrastructure need (signing, auth, secrets, deploy, caching, retries, queuing)
- The moment you are about to stage or commit an infra approach you invented on the spot

## When NOT to Use

- Genuinely novel or domain-specific work with no prior art to find. The scan returns nothing and you build
- A throwaway one-off where the first approach is cheaper than the scan and will never be maintained

## Watch-outs

- Timebox the scan so it does not become its own project. Its job is to surface the obvious better option, not to survey the field exhaustively
- Distinguish "cleaner in principle" from "cleaner for my constraints." Verify the found approach actually fits and actually delivers the result, for example that it lights the real Verified badge, not just that it claims to
- Prior art can be worse than your idea too. Weigh, do not cargo-cult the first thing you find
- Prior art includes the project's *own* decision docs, not just external search. The approach you are about to build may already be written down internally as the rejected one (an architecture doc, a tracked story). And the vendor's own docs often rank the options for you, read those before you design

## Adjacent Patterns

- `delegation-decision.md`, the sibling "pick the lightest build unit" call, applied to workers instead of infra approaches
- `grill-the-plan.md`, the adversarial pass that surfaces a plan's unstated assumptions before the ceremony locks them in
- `quick-win-scaffolding.md`, its counterweight, name mature patterns as opt-in seeds instead of building them prematurely

## Source

The personal agent session, 2026-07-11. Faced with Unverified commits from a keyless cloud agent, the plan was to build a setup that materializes a private SSH signing key from an environment variable and configures git to use it. That work was written and nearly staged. A two-minute scan of how ephemeral agents get Verified commits surfaced the platform-signing approach (create commits through the platform API, which signs them, no key at all), which was strictly better on every axis. The hand-built key path was abandoned before it shipped.

The household-agent repo, 2026-07-17. Migrating a secret into 1Password, the plan was to build a runtime resolver in the shared config module that shells out to `op read` per key. The prior art was in two places the design skipped: the repo's own `docs/secrets-architecture.md` had already chosen `op inject` at boot (render the env file from a template) and explicitly rejected the runtime-`op read` refactor, and 1Password's own docs rank the commands (`op read` for one-off use, `op inject`/`op run` for applications). Both the internal decision doc and the vendor guidance named the approach about to be built as the wrong one. Prompted by the operator asking to "search the web to make sure we're doing this well," which is the scan this pattern makes reflexive.
