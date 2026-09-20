---
type: pattern
date: "2026-06-28"
source: The personal agent Epic 4 (Story 4-2), building the portable worker-gate; verified against Claude Code headless hook behavior, 2026-06-28. Fills the gap flagged while hardening `thin-router-orchestrator.md` / `sensitivity-tiered-access-control.md`.
tags:
  - hooks
  - claude-code
  - subagents
  - layered-defense
  - security
---

# Gate Propagation into Headless Workers

You enforce a deterministic policy with `PreToolUse` hooks in your main agent. Then the agent dispatches work into another repo with a headless `claude -p`, and the policy silently stops applying — because the subprocess runs under the *target* repo's settings, not yours. The gates do not travel by default. Make them travel, explicitly, or every dispatch is an ungated hole in an otherwise airtight policy.

## The Problem

A `PreToolUse` deny hook is only as good as its reach. The natural assumption is that hooks are a property of "the agent," so a worker the agent spawns inherits them. It does not. A `claude -p` worker:

- loads the **target repo's** `.claude/settings.json`, not the parent's, so the parent's hooks are simply absent
- resolves `$CLAUDE_PROJECT_DIR` to the **worker's CWD**, so a hook command written as `$CLAUDE_PROJECT_DIR/hooks/deny.py` points at the wrong repo (or nothing)
- inherits no policy state unless you pass it in

So the most security-sensitive moment — handing a task to a worker that will touch a different domain — is exactly where the gates fall off, unless you carry them across the boundary yourself.

## The Pattern

**Ship a portable gate bundle with every dispatch, wired by absolute path and parameterized by environment.**

Four things make it hold (each verified against Claude Code behavior):

1. **Inject the hooks via `--settings`.** Pass a settings file (or inline JSON) on the dispatch that defines your `PreToolUse`/`PostToolUse` hooks. These DO fire in headless `-p` mode, and a `deny` cannot be bypassed even under `bypassPermissions`. Note that hooks **override per event, they do not merge** — your `--settings` PreToolUse fully *replaces* the worker repo's, which is what you want: your bundle becomes the authoritative gate.

2. **Reference hooks by absolute path, never `$CLAUDE_PROJECT_DIR`.** That variable resolves to the worker's CWD. Resolve the parent repo's hook paths at dispatch time and bake the absolute paths into the bundle.

3. **Pass policy roots through the environment.** The hooks need to resolve your policy file and your path map against the *parent* repo, not the worker's CWD. Set an env var (e.g. `POLICY_ROOT`) on the subprocess; parent env vars propagate into hook subprocesses. A second env var can switch the hook's *mode* per worker (see `router-worker-exfil-containment.md` — the worker's one authorized root).

4. **Fail closed at construction.** Build the bundle before spawning, and if any gate hook is missing, refuse to dispatch. A worker must never go out ungated because a file moved.

Set `disableAllHooks: false` in the bundle too, so a target repo that tries to opt out of hooks cannot.

## Why It Works

- **The gate is unbypassable where it matters.** A headless `deny` holds even under skip-permissions, so the worker is bound by the same deterministic policy as the parent — no honor system in the subprocess.
- **Absolute paths survive the CWD change.** The one thing that breaks naive propagation (path resolution against the wrong repo) is removed entirely.
- **Env carries the context, not the repo.** The same bundle works for any worker; what changes is the env (which policy, which authorized root), so there is one gate implementation, not one per worker.
- **Fail-closed construction** means the failure mode of a missing hook is "no dispatch," not "ungated dispatch."

## When to Use

- A router/orchestrator that hands work to headless `claude -p` workers in other repos (see `thin-router-orchestrator.md`)
- Any case where a deterministic `PreToolUse` policy must hold inside a spawned subprocess, not just the top-level session

## When NOT to Use

- In-process subagents that already run under the parent's settings (they inherit the hooks; no bundle needed). **But note what they do *not* inherit:** the parent's ability to *answer* a permission prompt. A command the parent can run by consenting is simply refused for them, silently — see `subagent-cannot-consent.md`. Hooks travel; consent does not.
- A worker that genuinely shares the parent's repo and CWD

## Watch-outs

- **Verify, don't assume, the headless hook behavior for your version.** The override-not-merge semantics and the `$CLAUDE_PROJECT_DIR`-resolves-to-CWD detail are the two that bite. Confirm them before relying on the bundle.
- **Scope `--allowedTools` too.** The bundle is a deny backstop, not the only control. A worker that has no `Bash` and no send tools in its allowlist cannot exfiltrate even if a gate has a gap. Belt and suspenders (see `router-worker-exfil-containment.md`).
- **Keep the bundle in sync with the parent's real gates.** If the bundle hand-lists hooks, a new gate added to the parent won't travel until the bundle is updated. Generate the bundle from the parent's gate set where you can.
