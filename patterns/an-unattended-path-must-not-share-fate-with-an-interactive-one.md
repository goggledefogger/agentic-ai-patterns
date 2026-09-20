---
type: pattern
date: "2026-08-05"
source: The personal agent session (2026-08-05) — a launchd voice-capture ping hung 16 minutes on `op read`; the 1Password CLI daemon was blocked on the desktop GUI, so every op call on the machine hung, including service-account reads that should never touch the app. Fixed by giving the unattended path its own $HOME.
tags:
  - reliability
  - automation
  - secrets
  - isolation
  - daemons
---

# An Unattended Path Must Not Share Fate With an Interactive One

A daemon, cron job, or launchd task that calls a tool a human also uses interactively (a secrets CLI, a cloud SDK, a git credential helper) usually shares that tool's per-user state: config files, sockets, keychains, session caches — everything keyed off `$HOME`. The interactive features ride that state, and when one of them wedges — a GUI waiting for a click, a biometric prompt with nobody at the keyboard, a helper daemon blocked on the app that spawned it — the unattended caller inherits the wedge. It does not fail. It waits, silently, forever, for a human who is not there.

## The failure that named it

The personal agent's voice-capture watcher transcribed a note in 4 seconds, then hung for 16 minutes on the *notification* — a `python` → `op read` fetching a Telegram token. The service-account token was present and valid; the read should never have prompted anyone. But the 1Password CLI routes through a per-user `op daemon`, that daemon integrates with the desktop app, and the app was stuck. Result: **every** `op` invocation on the machine hung — interactive, service-account, all of them — because they all met at one socket under `~/.config/op`. The hang held a lock file that Syncthing then replicated to three devices. `--config <dir>` did not escape it (the daemon socket still rides `$HOME`); a private `$HOME` did.

Two layers of the same lesson in one incident: the ping (interactive-adjacent, best-effort) also wedged the *watcher* (unattended, load-bearing), because they ran in one process chain with no bound between them.

## The Pattern

**Give the unattended path its own state, and its own clock.**

1. **Isolate the state.** Run the tool with a private, dedicated home (`HOME=~/.agent/op-home`, mode 700) or whatever fully separates its config, sockets, and caches from the interactive install. Test the isolation empirically — a tool may offer a config flag that moves *most* state while the blocking piece (a daemon socket, a keyring handle) still rides `$HOME`. The isolation that counts is the one that held up when the interactive side was actually wedged.
2. **Bound the time.** Every subprocess call on an unattended path gets a wall-clock timeout, sized generously (a false timeout that discards a slow-but-honest call costs more than a real hang caught a few seconds later). A hang must become a *named error* ("op waited for an interactive credential no unattended caller can supply"), because the error names the fix and a hang names nothing.
3. **Do not retry a prompt.** A timeout caused by an interactive wait is not transient — retrying it just multiplies the hang. Distinguish it from network flake in the retry logic.
4. **Keep best-effort steps from wedging load-bearing ones.** A notification, a metrics push, a cache warm — if it can block, it will eventually block the pipeline it decorates. Bound it separately or run it where its death is harmless.

## Why It Works

- **Shared state is shared fate.** The unattended caller was never using the desktop integration — it was only *standing next to it*. Separate homes make the dependency graph match the intent
- **A bounded hang is a diagnosable event.** 16 silent minutes produced nothing; a 10-second timeout with a named cause produced the diagnosis in one read
- **Not retrying prompts respects what the failure is.** The condition "a human is not here" does not improve with attempts

## When NOT to isolate

If the unattended path *deliberately* uses the interactive session (a user-present workflow, a tool that only works app-integrated), isolation breaks it — there the fix is only the timeout and the named error, so the wait is at least visible and bounded.
