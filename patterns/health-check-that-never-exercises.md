---
type: pattern
date: "2026-08-01"
source: The personal agent — walk-and-talk Kokoro synth answered /health with ok:true while every /speak returned 500 in 10ms, found 2026-08-01
tags:
  - monitoring
  - verification
  - liveness
  - anti-pattern
  - services
---

# A Health Check That Never Exercises the Work

A service exposes `/health`. It returns `{"ok": true}`. The process is running, the port is bound, the handler replied in a millisecond. Every one of those facts is true, and none of them is the question anyone is actually asking, which is: **can this thing do the one job it exists for?**

The gap between "the process is up" and "the work succeeds" is where a whole class of outages lives, and it is invisible precisely because the check is green.

## The Problem

`kokoro-server.py` holds a speech model resident so a walk does not pay a 46-second load per line. It exposes `/health` and `/speak`. On 2026-08-01 it had been started with a bare `python3` instead of the venv's interpreter, so `mlx_audio` was not importable.

The result:

- `/health` → `200 {"ok": true, "voice": "bm_george", "lines": 0}`
- `/speak` → `500 No module named 'mlx_audio'`, **in 10 milliseconds**

`/health` was honest about everything it measured. It reported the configured voice and the served-line count. It just never touched the model, because the model is only imported inside `synthesize()`.

The damage came from a *caller* that believed it. `kokoro-say.sh` gates on the health check before using the resident server:

```sh
if curl -fsS --max-time 1 "http://127.0.0.1:$PORT/health" >/dev/null 2>&1; then
  # ... use the fast path
fi
# ... otherwise fall through to the slow one-shot path
```

That gate is well-designed for the failure it anticipated — server absent. It has no defence against a server *present and useless*. So every line burned a round trip, fell through to a 5–8 second one-shot render, overran the client's audio budget, and the phone spoke in its own voice instead of the chosen one. For weeks that read as "the voice feature doesn't work," and every diagnosis started at the voice, which was fine, rather than at the health check, which was lying.

The tell was available the whole time and nobody read it: **`"lines": 0`**. The server's own counter said it had never synthesized anything. A health check that reports "I am fine" next to "I have never once done my job" is answering a different question than the one being asked.

## The Pattern

**A liveness check must exercise the work, or it must say it did not.**

Two ways, and the cheap one is usually enough:

**1. Prove it once at boot, then report that verdict.** The work is expensive; doing it per-request is silly. Do it once and let health carry the answer.

```python
_health_error = [None]

def main():
    try:
        synthesize("ready", DEFAULT_VOICE, DEFAULT_RATE)   # the actual job, once
    except Exception as exc:
        _health_error[0] = str(exc)                        # remember WHY
        print(f"warm failed: {exc}", file=sys.stderr)
    serve_forever()                                        # still serve — an honest 503 beats a closed port
```

```python
if parsed.path == "/health":
    if _health_error[0]:
        return self._send(503, "application/json",
                          json.dumps({"ok": False, "error": _health_error[0]}).encode())
    return self._send(200, "application/json", json.dumps({"ok": True, **_stats}).encode())
```

The caller's existing `curl -fsS` gate then works unchanged — a 503 fails it, and it takes the slow path deliberately instead of burning a round trip on a corpse.

**2. Expose the work counter and treat zero as suspicious.** `"lines": 0` on a server that has been up for an hour is not proof of failure, but it is a question worth surfacing. A dashboard that shows *jobs completed* next to *up* catches this class immediately.

## Why Keep Serving Instead of Exiting

Tempting to `sys.exit(1)` on a failed warm-up. Prefer staying up with an honest 503:

- The caller learns **why** (`No module named 'mlx_audio'`), not just "connection refused"
- A supervisor that restarts on exit will restart into the same broken environment forever
- Whatever cleans up the port (a tmux hook, a stop script) still finds a process to clean

An unhealthy service that can explain itself is worth more than a dead one that cannot.

## Related

- [[decorative-gate]] — a gate that cannot see what it claims to guard
- [[half-gate-whole-verdict]] — a check that proves one half and reports on the whole
- [[unrun-checks-read-as-passing]] — the same disease in test suites
- [[count-the-source-not-the-survivors]] — a correct count of the wrong population

## The Rule of Thumb

If the health check would still return `ok` with the core dependency uninstalled, it is checking that the process exists, not that it works. Name it `/ping` and stop calling it health — or make it do the job once and report what happened.
