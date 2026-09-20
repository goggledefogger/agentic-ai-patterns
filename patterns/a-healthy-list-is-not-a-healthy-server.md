---
type: pattern
date: "2026-09-09"
source: The personal agent voice pipeline, LM Studio serving chat from an engine-less process, measured 2026-09-09
tags:
  - local-models
  - reliability
  - infrastructure
  - lm-studio
---

# A Healthy List Is Not a Healthy Server

LM Studio answered `GET /v1/models` cleanly and returned an empty 404 on `POST /v1/chat/completions`, twice in a row. Every surface that looked like a health check said the server was fine: the port was open, the model list came back, the process was running. The one thing that actually mattered, whether a chat request got an answer, was never checked.

The server's own log showed the real shape of the failure. Every model-list call was logged and answered. Not one chat call reached the process that was doing the logging. Two processes owned port 1234 at once: the Bionic desktop app, launched first, and a separate headless `--run-as-service` process that started listening 27 seconds later. Whichever one macOS handed a given request to would run it or not, depending on which had an engine loaded, and the desktop app's copy had none.

## The Pattern

A model-list endpoint tells you a server process exists. It tells you nothing about whether that process, or a different process sharing its port, is the one that will answer the request you're about to send. When two processes can bind the same port at different times, the health surface everyone reaches for first (list the models, hit `/health`) can stay green while the actual work route is dead.

Before pointing a job at a local inference endpoint:

- **Confirm exactly one listener on the port.** `lsof -nP -iTCP:1234 -sTCP:LISTEN` should return one pid. Two means a desktop app and a background service are both up, and only one of them has an engine behind it. Quit the redundant one rather than guessing which will answer.
- **Probe the actual route the job will use, not a nearby one.** A one-word chat completion request, sent and checked before the real job starts, catches an engine-less server that a model-list call would never reveal. The two endpoints can diverge completely: one process can serve `/v1/models` off a config file it read at startup while the process that would run `/v1/chat/completions` is a different process entirely.
- **Keep a second endpoint to fall through to.** Ollama on its own port is a working fallback when the primary endpoint's chat route silently 404s. Don't retry the same broken endpoint hoping the second call behaves differently, it won't, the process behind it hasn't changed.

A second trap lives in the same surface: `/v1/models` lists every model LM Studio has ever downloaded, loaded or not, and a chat request against an unloaded one triggers a just-in-time load. Picking "the first id in the list" as a default can silently load a 12B model on a 16 GB machine mid-job. Query `/api/v0/models` and filter on `state == "loaded"` to pick from what's actually resident, or pick a known-small model by name rather than by list position.

The diagnostic move that actually found the split was reading the server's own request log, not guessing from symptoms. The log showed a clean stream of model-list calls and a total absence of chat calls, which rules out "the model failed to load" or "the request was malformed" in one look and points straight at "something else is answering." When a server's behavior contradicts what its cheap endpoints report, the log is the tiebreak, and it's usually sitting right there.

## Why the Model List Was the Wrong Thing to Trust

The model list and the chat route were, in this incident, answered by two different processes that happened to share a port and a config directory, so they agreed on what models existed while disagreeing completely on whether either could run one. A health check that only exercises the cheapest, most read-only endpoint on a server will always miss this class of split, because the split lives in which process handles which route, not in whether the server is "up" in any single sense.

The fix generalizes past LM Studio: any local server that can run as more than one instance, desktop app plus background service, dev server plus systemd unit, needs its liveness check to hit the actual work route, not the nearest cheap one.

## When to Use

Any time a job is about to send work to a local inference server (LM Studio, Ollama, or similar) rather than call it interactively and watch the reply land. This matters most for unattended or batch jobs, where nothing is watching the first response, and a silently 404ing endpoint would otherwise burn the whole batch before anyone noticed.

## Related

- [[no-delivery-without-arrival-accounting]], same shape one layer down: a channel, here a local inference endpoint, needs its own arrival signal, and a healthy-looking neighboring endpoint is not that signal
- [[count-the-source-not-the-survivors]], same family of false reassurance: a status that is accurate about the wrong thing looks identical to one that's actually fine

## Source

Measured 2026-09-09 in the personal agent's voice pipeline. `POST /v1/chat/completions` returned an empty 404 twice against LM Studio on port 1234, while `GET /v1/models` answered normally. The server log showed every model-list call logged and no chat calls reaching it. Two processes were bound to the port: the Bionic desktop app (launched 17:51:57) and a headless `--run-as-service` process (listening from 27 seconds later), and the engine-less desktop copy was the one answering chat. Fix: confirm a single listener with `lsof -nP -iTCP:1234 -sTCP:LISTEN`, probe the chat route with a one-word request before committing a job to an endpoint, and keep Ollama as a fallback.
