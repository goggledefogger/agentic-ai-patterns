---
type: pattern
date: "2026-10-02"
source: An agentic IDE harness whose shell execution demotes commands exceeding 5 s to background tasks; broad searches triggered task cancellations and silent turns
tags:
  - harness-interop
  - tool-use
  - async-tasks
  - shell-execution
  - silent-turns
---

# Bounded Searches Prevent Harness Async Demotion

An agentic model running across multiple harnesses often encounters different tool runtime mechanics. In one harness, a shell command runs synchronously until it completes or hits a long timeout. In another, any command taking longer than 5 s is demoted by the harness into a detached async background task.

When an agent needs to locate an unfamiliar file or project, its natural instinct is often to run broad shell searches (`find /workspace` or recursive `grep`). In a harness with async demotion, these broad commands exceed the threshold and spawn background tasks. Models frequently fail to navigate this lifecycle cleanly: instead of waiting or updating the user, they poll status, kill the background task, and terminate the turn with 0 output tokens. An upstream bridge nudging the wedged process gets no answer because the harness cannot accept subsequent prompts while in that state.

The prevention is to **bound exploratory commands before they reach the harness threshold**.

## The Pattern

- **Scope searches to shallow depths and targeted paths.** Never run unbounded recursive searches across entire home directories or large repository trees. Use explicit depth caps (`-maxdepth 2`) and target the specific directory where the file is expected
- **Prefer indexed tools over raw disk crawling.** Use fast version-control checks (`git ls-files`, `git log`) or semantic search tools rather than scanning cold filesystems with recursive shell tools
- **Explicit harness rules for non-native agents.** When loading operating instructions into a foreign harness, explicitly declare the threshold: commands exceeding 5 s switch to background tasks. Instruct the model never to kill background tasks and exit silently, and always to report partial findings to the user
- **Differentiate genuine background jobs from stalled queries.** Real background tasks (like long build scripts or dev servers) are intentional and belong in the background. Exploratory queries are synchronous requests that should fail fast or narrow their scope rather than transitioning into unmanaged background state

## Why It Works

Async state transitions in tool execution introduce complex state machines (task polling, cancellation notifications, detached streams). Large language models handle synchronous request-response tool calling far more reliably than managing async background task queues. Keeping exploratory tool execution under the harness threshold keeps the model on the primary turn-taking path where it reliably produces user-visible answers.

## When to Use

- When operating across different agent harnesses where tool execution models differ (such as IDE-integrated agents alongside CLI harnesses)
- When agents exhibit silent turn endings or wedged loops after running search commands
