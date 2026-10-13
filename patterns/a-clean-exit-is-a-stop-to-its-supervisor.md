---
type: pattern
date: "2026-10-13"
source: A dashboard restart drained the server cleanly, the service manager read the clean exit as "stay down", and the step that should have started the new copy died with the chat that ran it
tags:
  - lifecycle
  - supervision
  - restart
  - anti-pattern
---

# A Clean Exit Is a Stop to Its Supervisor

A service manager set to "relaunch on crash only" makes one quiet promise: exit with code 0 and you stay down. That is the right rule for a deliberate shutdown. It is the wrong rule the moment a *restart* is built out of a clean exit plus "something else will start the new one."

## The Problem

A long-running server sat under the OS service manager with a relaunch-on-failure policy. Its restart worked in two halves:

1. Signal the server. It drains open work and exits 0, a clean exit, by design.
2. The restart script, still running, starts the fresh copy.

The restart was run from a chat session that the server itself hosted. Half 1 ended the server, which ended the chat, which ended the script before half 2. The service manager saw a clean exit and did exactly what it was told: nothing. The user, away from the desk, found the dashboard gone with no error anywhere. The log's last line read "drained — down cleanly", which is true and is the problem.

Two properties made it invisible:

- **Every piece was correct alone.** The drain was graceful, the exit code honest, the supervisor policy reasonable. The bug only exists in the hand-off between them.
- **"Restart" and "stop" produce the same exit.** The supervisor cannot tell them apart, so whichever process was meant to carry the restart is a single point of failure, and it is usually a child of the thing being restarted.

## The Pattern

- **If the service is meant to always be up, tell the supervisor so.** Relaunch on *any* exit. Turning it off becomes an explicit act on the supervisor (unload the job), never a side effect of a clean exit
- **Make the restart a hand-off to the supervisor, not a child process.** The restart drains the old copy and asks the supervisor to start (or confirm) the new one. A copy started by the caller dies with the caller, and the supervisor reports the job as not running while it serves
- **Give the supervised launch a standby mode.** If another launcher (an in-app restart, a dev supervisor) can also bring the server up, the always-relaunch job must wait while a healthy copy answers and take over when it leaves. Otherwise it crash-loops on "port in use" every throttle interval
- **Verify by stopping it cleanly.** The test is not "kill -9 and watch it return"; that only proves the crash path. Send the polite stop and watch for a new process id, owned by the supervisor

## Why It Works

The supervisor is the only process guaranteed to outlive the restart. Anything else that carries the "start it again" step can be taken down by the very stop it follows. Moving that step into the supervisor removes the gap between the two halves instead of trying to make the caller survive it.

## When to Use

- A daily-driver service under launchd, systemd or a process manager with an on-failure policy
- Any restart triggered from inside the thing being restarted: a chat it hosts, a button it serves, a hook it runs
- Two launchers for one port

## Related

- [The Teardown Was Verified. The Startup Was Never Written.](teardown-verified-startup-never-written.md): the same blind spot, one level down
- [Deployed Is Not Loaded](deployed-is-not-loaded.md): after a restart, the evidence is a new process id
