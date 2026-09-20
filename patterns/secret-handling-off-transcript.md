---
type: pattern
date: "2026-07-17"
source: The household-agent repo (a household agent on a Raspberry Pi), migrating the Sonarr API key from plaintext .env into the 1Password household vault, 2026-07-17
tags:
  - security
  - secrets
  - agents
  - workflow
---

# Secret Handling Off-Transcript

When an agent has to move or provision a live secret value, two things must hold: the least-privileged automation credential does not get widened to do the write, and the secret value never lands in the durable transcript. The agent transcript outlives the task. A key echoed into it is a leak with a long tail, in a place `.gitignore` and file permissions do not reach.

## The Problem

An agent wiring up a secret hits two temptations. First, the automation credential it already holds is read-only (a service-account token, a deploy key), so the reflex is to widen its scope so the agent can write the new secret into the vault itself. That trades away the least-privilege boundary for convenience. Second, to move the value from where it lives (a plaintext `.env`) into where it belongs (the vault), the agent reads it, and the natural way to read a value is to print it, or to pass it as a command argument. Either one deposits the secret into the transcript or the process table.

Both are avoidable. The read-only credential being unable to write is the feature, not the obstacle. And the value can travel machine-to-machine without ever passing through the agent's output.

## The Pattern

**Route the write through the human's session, not a wider token.** A read-only automation credential can't provision secrets by design. Don't grant it write scope. Do the one-time write through the human's own write-capable session, their interactive sign-in, their Touch ID approval on their own machine, and leave the automation token read-only. Provisioning is rare and human-authorized; the runtime stays least-privilege.

**Keep the value off the transcript and off argv.**

- Never print it. Verify by hash or length comparison, never by echoing the value (`sha=fa652ccb len=32`, not the key).
- Feed it via stdin or a temp file that is never printed, never as a command-line argument. Argv is visible to `ps` and, worse, appears in the command you echo. Process substitution (`field=$(ssh host 'cat secret')`) still expands into argv, use a temp file redirect instead.
- Move it directly source to destination so it transits a pipe, not your output. Only the final tool's stdout is captured; a piped intermediate value and a concealed vault field both stay out of it.
- Dry-run writes to a password manager first (`op item edit ... --dry-run`), confirm the shape, then commit.
- The shell `!`-prefix / echoed-command path is the trap: it lands the value in the visible transcript. A plain piped tool call does not.

## Why It Works

- The durable artifacts, transcript and git history, never contain the secret. Exposure collapses to a transient, local, human-owned moment: the value in a temp file on the human's own machine for a second, then deleted.
- Least privilege survives contact with the task. The automation credential that runs unattended forever stays read-only; only the human's rare, interactive session can write.
- Hash-verification proves the migration without revealing anything. `op://vault/item/field` resolving to the same 12-char SHA the live value hashes to is full proof that the reference is correct, printed safely.

## When to Use

- Any time an agent provisions or moves a secret: seeding a vault, rotating a key, wiring a new integration, backing up an env file.
- Especially when the agent holds a read-only deploy or service credential and a write is needed. The split (read-only automation + human write session) is the whole point.

## When NOT to Use

- Pure read paths where no value ever moves. Resolving a secret at runtime through a reference (`op read op://...`) never materializes it in your output and needs none of this.
- When the human just does the write in the GUI. That is also correct, the discipline is that the secret never enters the agent's durable record, by whatever route achieves it.

## Watch-outs

- Off-argv beats off-transcript, but do both. A temp-file source keeps the value out of your typed command; leaving it in argv still exposes it to local `ps`.
- Verify the write from the least-privileged consumer, not the write session. Prove the read-only token that will actually run can resolve the new reference, not just that the privileged session can.
- A provenance-only migration (plaintext to vault-rendered) should change nothing. Prove that, do not trust it: render and byte-diff against a pre-change backup, a mismatch means you changed the value, not just its source. Pair the new indirection with a fail-open path (leave the last-good file, exit 0) so the migration can't harden into a boot-time outage. See `atomic-state-writes.md` for the temp-file + rename mechanic underneath.

## Adjacent Patterns

- `scan-prior-art-before-building-infra.md`, the sibling scan that picks the right secrets mechanism in the first place (`op inject` at boot over a hand-built runtime resolver)
- `sensitivity-tiered-access-control.md` and `router-worker-exfil-containment.md`, protecting secret *content the agent reads*; this pattern protects a secret *value the agent moves*
- `atomic-state-writes.md`, the temp-file + rename the fail-open renderer is built on

## Source

The household-agent repo, migrating `SONARR_API_KEY` from plaintext `~/.hermes/.env` into the 1Password household vault, 2026-07-17. The Pi's service-account token is read-only, so the vault write went through the operator's personal `op` session on their Mac; the key was fetched Pi to a local temp file (never printed), added with `op item edit --dry-run` then committed, and verified from the Pi's read-only token by SHA-comparing `op://the household agent/Sonarr Media Server/api_key` against the live `.env` value, all without the 32-character key ever entering the transcript or a command line.
