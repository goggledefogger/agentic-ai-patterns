---
type: pattern
date: "2026-06-28"
source: The personal agent Epic 4 (Stories 4-3/4-4 + the read-deny scope addition), passing the Switch 2 adversarial prompt-injection test, 2026-06-28. The exfil-side companion to `sensitivity-tiered-access-control.md`, which covers the write/read-tier gate. REVISED 2026-07-16 after the personal agent's router read-deny was measured against six ordinary ways to read a protected file and lost three — found by accident, while diagnosing something unrelated, on a hook that had passed its adversarial suite. The router bullet was refuted by this pattern's own worker reasoning, which it had carried since 2026-06-28 without noticing.
tags:
  - security
  - privacy
  - hooks
  - exfiltration
  - layered-defense
  - routing
---

# Router/Worker Exfiltration Containment

A tier-path deny that only blocks *writes* leaves the data-out door open. Two moves close it: the router reads nothing sensitive (it routes), and a worker may read its one authorized root but can never send. Together these mean sensitive content has no path from disk to the outside world that a prompt injection can walk.

## The Problem

`sensitivity-tiered-access-control.md` stops an agent *writing into* a protected root. But "keep secrets in" has a second half the write-gate misses: **exfiltration**, the secret flowing *out*. Two exits a write-only gate ignores:

1. **Read-then-send.** A prompt injection tells the agent to `cat` a secret into its own context, then email/POST/commit it somewhere public. Nothing was written to a protected path; the read was the leak.
2. **The shell and the send tools.** A `>` redirect resolved from `~` or a relative path, a `curl --data @secret`, an MCP `send`/`upload` — none of these look like an `Edit` into a protected root, so a path-keyed write-gate waves them through.

You cannot reliably detect "this text is sensitive" at send time (that needs taint-tracking). So contain the exits structurally instead.

## The Pattern

**Make the router incapable of holding sensitive content, and the worker incapable of sending it. Then there is nothing to taint-track.**

Two roles, one env-parameterized deny hook:

- **Router mode (no authorized root set).** The front-door agent *routes*; it never touches sensitive content directly. The deny hook blocks **reads as well as writes** of every protected root — `Read`, `Grep`, and shell reads (`cat`, redirects) included. The router literally cannot pull a secret into its context, so the read-then-send exit is closed at the read. **The shell half of that sentence is aspirational unless the OS enforces it — see "The router's shell read-deny is not a boundary" below. Pair this with `sandbox.filesystem.denyRead`; the hook alone does not deliver what this bullet promises.**
- **Worker mode (exactly one authorized root, passed by env).** A dispatched worker owns one protected root: it may read and write *that* root (its job) and nothing else sensitive. But it **cannot send** — every egress tool is denied, and it runs **no shell at all** (deny `Bash` wholesale; a binary-name denylist over a shell string is always bypassable, so don't try). The worker returns a *summary* to the router, never raw content and never an outbound message.

Gate the egress surface **tier-aware**, mirroring the routing key:

- personal/private content in an egress payload → **deny**
- co-owned / shared-IP content → **ask** (a human "is this mine to share?" pause)
- the egress tool set is open-ended (MCP send/upload, web POST, messaging, browser navigate), so match inclusively and only act on a real sensitive reference

Resolve shell path tokens properly: expand `~`, resolve relative and `..`, compare resolved paths boundary-safe (not `startswith`). A substring match over the raw command misses `~/secret` and over-matches siblings. **Necessary, but do not mistake it for sufficient — this line read as a recipe for a closed boundary and it is not one. See below.**

## The router's shell read-deny is not a boundary

**Added 2026-07-16, from a live failure in the personal agent.** This pattern already contains the argument that refutes its own router bullet, and did not notice: *"a curated denylist of egress binaries is not [fail-closed] (think `bash -c`, `python -c`, `$(...)`, encodings). Prefer the wholesale deny."* That reasoning was applied to the **worker** (deny `Bash` outright) and never carried back to the **router**, whose shell read-deny is exactly the thing it calls always-bypassable. The router's deny is path-keyed rather than binary-keyed, which is more tractable — but only for paths a parser can see.

The personal agent's `tier-path-deny.py` does everything the line above prescribes: `shlex` tokenization, `~` expansion, relative/`..` resolution, boundary-safe comparison. Measured against six ways to read the same protected file, 2026-07-16:

| Command | Result |
|---|---|
| `cat /root/x.md` | **DENY** |
| `cd /root && cat x.md` | **DENY** |
| `grep -r secret ~/root` | **DENY** |
| `d=/root; cat "$d"/x.md` | *allow* — assignment resolves to `<cwd>/d=/root`, so the root never appears as a token |
| `cat $(echo /root)/x.md` | *allow* — the path does not exist until the shell runs |
| `python3 -c "print(open('/root/x.md').read())"` | *allow* — the path is inside a program string |

**Three of six, and the leaks are not the adversarial ones.** `python3 -c` is what an agent reaches for on an ordinary task; a variable assignment is what it writes to keep a command readable. The gate holds against the shapes you would type while *thinking about* the gate, and fails against the shapes you type while thinking about the work — the inverse of what a control should do, and the reason this went unnoticed: the deny fires often enough to feel real. **A control that passes every test you think to write against it, and fails the commands you actually run, is worse than none — it buys confidence it has not earned.** the personal agent's own `CLAUDE.md` claimed "the binding stop is the deterministic deny hook," and that sentence was true only for the structured tools.

**The fix is a layer, not a better parser.** `sandbox.filesystem.denyRead` (macOS seatbelt / Linux bubblewrap) refuses the `open()` regardless of how the path was spelled, and covers child processes — so all six rows deny without parsing anything. Keep the hook: on `Read`/`Grep`/`Edit`/`Write` it inspects a real path field, is exact, and is the only layer that knows the *tier* and can name the authorized route. **Rank them honestly rather than blurring them together:**

| Layer | Strength | Against |
|---|---|---|
| OS sandbox (`denyRead`) | a boundary | anything, including code the agent writes |
| `permissions.deny` `Read(...)` | a rule | built-in tools + recognized bash file commands |
| tier-path deny hook | exact on structured tools, **a speed bump on Bash** | accident and honest error, not indirection |
| agent honor system | weakest | forgetting only |

The hook is not demoted to theater — it is the layer that makes a denial *legible* and routes the work to an authorized worker. It is demoted from "boundary" to "the tier-aware layer on top of the boundary." Say which one you have.

## Why It Works

- **No content to leak.** The router never has sensitive bytes in context, so there is nothing for an injection to forward. The classic read-then-send attack has no source.
- **No channel to leak through.** The worker that *does* hold sensitive bytes has no send tool and no shell, so it has no outbound channel. The summary it returns is the only thing that crosses back, and it is content the worker chose to surface, not a raw dump.
- **Bypassable controls are removed, not patched.** Denying all worker `Bash` is fail-closed; a curated denylist of egress binaries is not (think `bash -c`, `python -c`, `$(...)`, encodings). Prefer the wholesale deny.
- **The human stays in the loop only where judgment is needed.** Hard-deny the unambiguous (private content out); ask on the genuine judgment call (co-owned IP).

## When to Use

- A `thin-router-orchestrator.md` over mixed-sensitivity domains, where the threat model includes prompt injection from hostile downstream content
- Any agent that both reads protected data and has tools that can reach the network or another person

## When NOT to Use

- A single-tier, single-user, offline workflow with no egress tools — there is no exit to contain
- A worker whose entire purpose IS to publish (e.g. `vault-as-cms-publisher.md`) — that send capability is the point, and is gated by a different, deliberate authorization, not denied

## Watch-outs

- **Read-deny is router-only.** If you put it in the portable worker bundle too (`gate-propagation-into-headless-workers.md`), the worker can't do its job. Parameterize by the worker's one authorized root.
- **Residual risk: semantic leakage in an allowed summary.** A worker summary that legitimately returns and happens to embed a secret, then is sent by the router, is not caught — the gates key on paths and tools, not on meaning. The structural guarantees keep this path narrow; content taint-tracking is the upgrade if it ever bites. Document it; don't pretend it's closed.
- **Prove it adversarially.** Plant an injection in a sensitive fixture and assert every induced call (write, read, redirect, send) is denied or paused, for both roles. A deterministic gate-level test runs in CI; an LLM-driven one does not.
- **Then prove it *ordinarily* — this is the one that caught the personal agent.** Adversarial tests are written by the person who built the gate, so they probe the shapes that person imagined; the personal agent's gate passed its adversarial suite and lost to `python3 -c`. Take the last 20 Bash commands from a real transcript, point them at a protected root, and assert every one denies. Ordinary commands are the ones an agent actually runs, and they are drawn from a distribution you cannot invent by trying to think like an attacker.
- **State the strength of each layer in the code that implements it.** The gap here was never that anyone believed the parser was perfect — it is that "the deny hook is the binding stop" got written once, in a doc, unqualified, and every later decision quietly inherited it. Rank the layers where a reader will meet them.
