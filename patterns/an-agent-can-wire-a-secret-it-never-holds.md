---
type: pattern
date: "2026-08-12"
source: The personal agent — onboarding a revived Telegram bot token; the author offered to paste it into the chat, and an earlier bot token echoed into a session transcript is still awaiting rotation because of exactly that move
tags:
  - secrets
  - agents
  - op
  - transcripts
  - workflow
---

# An Agent Can Wire A Secret It Never Holds

An agent session is a recording device. Everything pasted into it — chat,
tool output, command lines — lands in a transcript that outlives the moment,
gets synced, summarized, and read back by later sessions. So the naive flow
("paste me the token, I'll store it") converts a secret into a log line. The
fix is not "be careful": it is a division of labor where **the agent composes
the machinery and the human supplies the value on a channel the session
cannot see.**

## The Problem

Onboarding a bot token (2026-08-12): the operator, reasonably, offered to
paste it into the chat for the agent to store in 1Password. The fleet's own
registry records why that fails: a previous bot token was echoed into a
session transcript once, and that bot has carried a "rotation pending" flag
ever since — the paste took seconds, the cleanup is still unpaid. A secret
that transits the conversation is published to every future reader of that
conversation.

The tempting alternatives each leak somewhere quieter: the agent putting the
value in a command's arguments logs it in the tool call; `echo`-ing it to
verify prints it; shell history keeps whatever was typed as an argument.

## The Pattern

The agent builds and hands over a one-liner; the human runs it where the
value stays theirs:

```bash
read -rsp "token: " T && echo && \
  op item create --vault <Vault> --title <item> --category "API Credential" \
    "token[password]=$T" >/dev/null && unset T && echo stored
```

1. **Hidden prompt, human's own terminal.** `read -rs` masks the paste; the
   value enters a variable, not a typed argument — so no chat, no tool
   output, no shell history. (A clipboard pipe — `pbpaste | …` straight into
   the store — is the same move when a prompt isn't available.)
2. **The agent sees only "stored".** Output is suppressed or reduced to a
   verdict. The agent orchestrated everything and held nothing.
3. **Validate shape without printing.** If the flow can check the value
   (regex for a token format), it prints MATCH/NO MATCH — never the value,
   not even on failure.
4. **Verify by fetch-through-the-sanctioned-path, measured not shown.** The
   agent confirms the secret landed by fetching it through the same route
   production will use and printing only its length. Existence and shape,
   never contents.
5. **Clear the residue.** `unset` the variable; clear the clipboard if one
   was used.

Known ceiling, named rather than hidden: the value briefly appears in the
storing process's argv (`ps` on the same machine could catch it). On a
single-user box that is accepted; where it isn't, feed the store from stdin.

## The Whole Chain, Audited (asked and answered 2026-08-12)

"Is this truly secure at every step?" No flow is; a trustworthy one names its
residues. For this run: **transcript clean** (the design goal, achieved);
**clipboard** held the value before the hidden prompt — clipboard history and
macOS Universal Clipboard can retain/sync it, so overwrite or clear it after;
**terminal scrollback** holds anything later *displayed* for re-pasting (a
wizard prompt) — clear that window; **argv blip** during the store, accepted
on a single-user machine; and the receiving tool may write its own **resting
plaintext copy** (this one saved the token into its config file on the target
box — a second home the vault doesn't govern).

The production-grade variant closes the resting copy: the service fetches the
secret from the vault at start (scoped service account) and hands it to the
process — env or a private runtime file — so disk never holds a standing
copy, and the vault remains the only place a rotation has to reach. And
verify what child processes inherit: a token in the service's environment is
readable by everything the service spawns unless deliberately dropped.

## When It Applies

Any time an agent workflow needs a credential to exist somewhere — vault
item, env file, CI secret — and the human has it on screen. The agent's
contribution is the exact command, the validation, and the verification;
the human's is one paste into a surface that doesn't record.
