---
type: pattern
date: "2026-08-08"
source: The personal agent cloud-VM leader — "outbound HTTPS is broken"; actually one IPv6 path to one host resetting during the TLS handshake, ~25% of calls, invisible to every dual-stack fallback on the box, 2026-08-08
tags:
  - debugging
  - networking
  - verification
  - anti-pattern
  - reliability
---

# The Fallback Committed Before the Failure Happened

A scheduler's push leg failed. A Telegram poller left updates sitting server-side. Both died with `[Errno 104] Connection reset by peer`, and the box had a plausible story ready: outbound HTTPS is broken.

It was not. `curl https://api.telegram.org` returned 302 in 470ms. So did the IPv6 form. So did a second host on both families. Four green probes, and the production error was still there in the journal every twenty minutes.

The gap between those two facts is this pattern.

## The Problem

`api.telegram.org` had one A record and one AAAA record. The IPv4 path was perfect. The IPv6 path reset **during the TLS handshake** about a quarter of the time. `getaddrinfo` returns the AAAA first under RFC 6724, so that was the path nearly every client took.

Now the part that matters. Both clients on the box have address fallback, and neither one helped:

- **curl 7.88 implements Happy Eyeballs** (RFC 8305). It races the IPv4 and IPv6 TCP connects and takes whichever answers first.
- **Python's `socket.create_connection` loops over every `getaddrinfo` result**, trying each in turn and moving to the next on `OSError`.

Both mechanisms select an address **at TCP connect time**. The v6 connect succeeded — 0.158s, clean, no error to trigger anything. The reset arrived one layer up, inside `ssl.do_handshake()`, after the socket was already chosen and the fallback loop had already exited:

```
File "http/client.py", line 1474, in connect
    self.sock = self._context.wrap_socket(self.sock, ...)
File "ssl.py", line 1379, in do_handshake
    self._sslobj.do_handshake()
ConnectionResetError: [Errno 104] Connection reset by peer
```

`HTTPSConnection.connect()` calls `create_connection()` — the loop with the fallback — and *then* wraps the socket in TLS. The failure is outside the loop. The retry that exists cannot fire, because from its point of view nothing failed.

Measured, same box, same minute:

| client | rounds | failures |
|---|---|---|
| bare `curl` (Happy Eyeballs active) | 20 | 7 |
| stock `urllib.request.urlopen` | 20 | 8 |
| `curl -4` / IPv4-pinned Python | 79 | 0 |
| `curl -6` / IPv6-pinned Python | 79 | 15 |

The client engineered to survive a bad address family failed at the same rate as the naive one. That is the whole pattern in one row.

## The Pattern

**A failover mechanism protects the layer it makes its choice at, and no layer above it.** Ask of any fallback: *at what moment does it commit, and what can still go wrong after that moment?* Everything after the commit point is unprotected, no matter how robust the mechanism looks.

This is not a networking quirk. The same shape, everywhere:

- A load balancer health-checks a TCP port; the app behind it returns 500 to every real request. The pool never marks it down.
- DNS failover pings the host; the host answers ICMP and serves an expired certificate.
- A connection pool validates with `SELECT 1`; the real statement dies on a lock or a permission.
- A retry wrapper catches `ConnectionError` but the service returns `200 OK` with `{"ok": false}` in the body.
- A CI job retries on non-zero exit; the flaky step exits 0 and writes a corrupt artifact.

In every case something upstream said "this endpoint is fine" using a cheaper signal than the one the caller actually depends on, then handed over a committed choice.

The tell in review: a fallback, retry, or health check whose success condition is **strictly cheaper** than the work it is standing in for. Cheaper is the point — that is why it is fast — but it also fixes exactly how far up the stack the protection reaches.

## How You Actually See It: One Probe Is Not a Rate

The fault sat at roughly 25%. A single `curl` passes three times in four. Four single probes across a 2×2 of host × family — precisely the right *design*, run once per cell — had a better-than-even chance of coming back all green and sending the investigation somewhere else entirely.

An intermittent fault is a **rate**, and a rate needs a sample:

- **Hold one variable, repeat.** 15 rounds per cell turned "all four work" into `v6 telegram: 11/15` while the other three cells stayed perfect.
- **Keep the negative control in the matrix.** IPv6 to a *different* host was 15/15. That is what upgraded the finding from "IPv6 is broken" (wrong, and would have justified disabling it host-wide) to "this one v6 path is broken" (right, and fixed with four lines).
- **Compare the rate to production.** The measured ~25% matched 49 resets across 198 journal runs over 36h. When your bench rate matches the observed rate, you are looking at the actual fault and not a second one.
- **Re-run the failing leg after the fix.** IPv6 was still failing 9/40 afterward. The fault did not go away; the code stopped walking into it. If the broken leg goes green on its own, you fixed nothing and the outage will be back.

A single green probe and a 25% failure rate are the *same observation* most of the time. Treat one pass as one sample, not as a verdict.

## Why It Works

- Naming the commit point turns "why didn't the retry help?" into a question with a checkable answer, instead of a mystery that gets written off as flakiness.
- The cheap-signal test is greppable in review. A health check that returns in a millisecond is not exercising a model load, a disk write, or a TLS handshake — and you can say so without running anything.
- Sampling per-variant is what separates "them", "us", and "the path between us". One probe cannot make that split, and the split is the whole diagnosis.

## When to Use

- Any report of the form "it works when I test it but fails in production." Check whether your probe and the program commit at the same layer, and sample before concluding.
- Before trusting a retry, failover, or health check you did not write. Find the commit point.
- Whenever a fix consists of pinning a transport, an address, a region, or a replica: verify the *unpinned* path is still broken afterward, or you have not shown your fix did anything.

## When NOT to Use

- A hard-down dependency (connection refused, NXDOMAIN, 100% failure) needs one probe, not thirty. Sampling is for faults with a rate; deterministic faults announce themselves on the first try.
- Do not reach for the host-wide knob when the matrix says one cell is bad. `/etc/gai.conf` precedence would have "fixed" this by deprioritising IPv6 for every process on the box, to route around one prefix on one host — and it would have left no trace in the repo for the next machine.

## A Note on the Probe That Lied Twice

The same session produced a second, cheaper instance worth naming, because it is the one that nearly closed the investigation early. Checking for recently-written files:

```
find . -newermt '2 hours ago' -type f 2>/dev/null
```

Empty output, read as "nothing was written." The box ships `bfs` as `find`; it rejects `-newermt '2 hours ago'` as an invalid timestamp and exits with an error — which `2>/dev/null` swallowed. Files written 27 minutes earlier were sitting right there.

**Never discard stderr on a command whose empty output you intend to read as evidence of absence.** The suppression that keeps output tidy is the same suppression that converts "this command did not run" into "there is nothing there," and absence is the one claim that looks identical either way. If empty output would change your conclusion, show the exit code and the errors.

## Adjacent Patterns

- `ack-outran-the-write.md`, what this fault was hiding. The reset was the trigger; a cursor that advanced before its write was the defect, and it cost an unrecoverable voice note. Pinning the transport here dropped the trigger rate to zero, which would have buried that bug intact — when you fix a trigger, go looking for what it was exposing.
- `health-check-that-never-exercises.md`, the same commit-point defect in monitoring: `/health` returns ok because it answers a cheaper question than `/speak` does. This pattern is the general form; that one is the liveness instance.
- `verification-needs-a-negative-control.md`, the control leg is what made the IPv6-to-Cloudflare cell load-bearing here. Without it the finding is "IPv6 is broken" and the fix is a sledgehammer.
- `decorative-gate.md`, a fallback that cannot fire is a gate that cannot refuse, one layer down.
- `plausible-cause-ends-the-search.md`, "outbound HTTPS is broken" explained every symptom and was wrong; a story that fits is not a story that was tested.
- `an-event-is-not-a-cause.md`, the resets were real and the conclusion drawn from them was not.

## Source

The personal agent cloud-VM leader, 2026-08-08. `the personal agent-scheduler`'s nudge-push and the `the personal agent-telegram-gateway` poller both failing with `[Errno 104] Connection reset by peer` while every hand-run `curl` came back clean. Root cause: the IPv6 path to `api.telegram.org` reset during the TLS handshake on ~25% of attempts; `getaddrinfo` returns the AAAA first, and both curl's Happy Eyeballs and Python's `create_connection` fallback loop had already committed to that address at TCP connect time. Fixed by pinning the four Telegram call sites to IPv4 via a `scripts/ipv4_https.py` opener, rather than the host-wide `/etc/gai.conf` precedence line, so the fix travels with the repo and stays scoped to the one broken path.
