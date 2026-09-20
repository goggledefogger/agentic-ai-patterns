---
type: pattern
date: "2026-08-22"
source: The household-agent repo (a household agent on a Raspberry Pi), Hermes gateway prompt caching, agent board #56
tags:
  - deployment
  - agents
  - verification
  - caching
  - anti-pattern
---

# Deployed Is Not Loaded

A deploy changes bytes on disk. Whether the running system ever reads those bytes is a separate question, owned by the consumer, and a long-lived process usually answers it once — at startup — and never again.

## The Pattern

A rule was added to an agent's instruction file: property handoffs must pass `--requester`, so the finished report has somebody recorded to send it to. It was committed at 05:04Z, deployed to the Pi at 06:07Z, and verified byte-identical against the repo. Forty minutes later the agent opened a job without the flag, using the pre-deploy form of the command verbatim. The job cannot be delivered: the downstream poller refuses to guess who a house report goes to.

Nothing failed. The file was correct on disk, the deploy exited 0, `gh` exited 0, and the agent's own reply ("handed it off, I'll send you the page") was true of the command it ran.

The gateway keeps one long-lived agent object per chat and captures the system prompt into it exactly once:

```python
# agent/turn_context.py
if agent._cached_system_prompt is None:
    restore_or_build_system_prompt(agent, system_message, conversation_history)
```

`_cached_system_prompt` is set on the first turn and is never `None` again. The session had started sixteen hours before the rule existed, so it was running a sixteen-hour-old prompt and would have kept doing so for as long as the conversation stayed alive — through any number of redeploys.

The deploy had a step for exactly this. It blanked the JSON session map, which a storage migration two months earlier had turned into a fossil (per-session files in that directory stop dead on the migration date; the live store is SQLite). It wrote `{}`, printed `Reset 0 session(s)`, and moved on.

So there were two independent failures stacked, and the second is the more interesting one:

- The actuator wrote to a store the migration had abandoned — the writer-side twin of [[migration-blinds-readers]], where readers of the old store return empty and you trust the emptiness.
- **Writing to the correct store would not have helped either.** The stale rule was not in a file. It was in a cache inside a process. No file an outside deploy can write reaches it. The only lever is killing the process.

## Why It Works

Ask of any deployed config: **who reads this, and when do they read it?** "On every use" and "once at startup" are wildly different systems, and the second is the common one — nginx workers, JVM config singletons, feature-flag caches, module-level constants in a worker, a DNS answer inside its TTL, an agent's system prompt.

For anything cached at startup, **the reset is a restart, and the evidence is an identity change** — a new PID, a new worker generation, a new connection epoch. Not a file's contents, and never the actuator's own say-so:

```bash
PID_BEFORE=$(systemctl --user show "$SVC" -p MainPID --value)
restart
PID_AFTER=$(systemctl --user show "$SVC" -p MainPID --value)
[[ "$PID_AFTER" != "$PID_BEFORE" && -n "$PID_AFTER" && "$PID_AFTER" != 0 ]] || fail_loudly
```

Same PID means the process holding the stale config is still the one answering. That is not a reset, and reporting it as one is how sixteen hours of drift stays invisible.

Two habits fall out of the second failure:

- **A reload step must name what it is reloading, in the same words as the thing that holds the state.** "Reset sessions" described a file. What needed resetting was a cache in a process. The step read as correct for years because nobody re-asked the question after the storage moved underneath it.
- **Put the refusal where the missing input is still recoverable.** The handoff took an optional `--requester` and quietly opened a job without one; the complaint arrived much later, on another agent's side, where the requester's name no longer existed to be recovered. Making the flag mandatory for that lane turns a silent hole into a non-zero exit in front of the caller — which also makes the stale-prompt case *self-announcing*, because an agent working from an out-of-date instruction file now hits an error that tells it to go re-read the file.

## The Tell

The failure produces no error anywhere, so the only signal is behavioral: **something ran an older version of a procedure than the one on disk.** Read the transcript for the shape of the command, not the exit code. A command that matches a form you retired is a cache you have not invalidated.

When checking whether a shipped instruction actually took effect, the file's mtime and checksum prove delivery to the disk and nothing else. The question is always whether the consumer re-read it, and the answer is in the consumer's lifecycle, not in the file.
