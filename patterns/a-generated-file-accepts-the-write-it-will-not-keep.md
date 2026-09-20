---
type: pattern
date: "2026-08-17"
source: The household-agent repo / the household agent (Pi agent) — BOARD_TOKEN hand-appended to the rendered ~/.hermes/.env twice, destroyed twice by the next gateway restart's re-render; the second loss took the agent-to-household-agent task channel down for 11 silent hours
tags:
  - config-as-data
  - drift
  - secrets
  - generated-files
  - anti-pattern
---

# A Generated File Accepts the Write It Will Not Keep

When a file becomes a render target, it does not stop being writable — it stops being *durable*. Every editor, `>>`, and runbook that touched it before the migration still works, still exits 0, and still verifies: the key is right there in the file, the consumer reads it, the task completes. The write is not rejected; it is scheduled for deletion at an unrelated moment. The next render — a service restart, a deploy, a boot — replaces the file wholesale, and the hand-added line is gone with no error, because from the renderer's point of view nothing is wrong: it produced exactly what the template says.

This is worse than a read-only file. A read-only file fails the write at write time, when the person who knows what they were doing is watching. A generated file fails it at render time, when nobody is.

## The Problem

A Pi agent's `~/.hermes/.env` had been the canonical secrets file for months. Then a hardening story made it a render artifact: a template (`.env.tpl`) holding 1Password references, an `op inject` script as `ExecStartPre` on the gateway service, a full `mv tmp .env` overwrite on every start. Sound design — rotation became "update 1Password, restart."

Three weeks later a new credential arrived: a GitHub PAT for a task-polling cron. It was appended straight to `.env`, where every credential before the migration had gone. Verified working. Some restarts later: gone. Appended again — the second write is the tell that nothing about the first loss taught anything, because nothing *surfaced* the first loss. Gone again after the next deploy's restart. The poller then refused to run (correctly — its fallback would have been a far more privileged ambient login) every 10 minutes for 11 hours, into a log nothing read.

Three design choices compounded, each individually reasonable:

1. **The render was fail-open and quiet.** Right call for boot resilience — a 1Password hiccup must not block the gateway. But it means the overwrite path has no failure mode a human ever sees.
2. **The render's sanity gate validated only what the template knew about.** It checked that every *reference* rendered non-empty. A key the template never contained is invisible to it — the gate protects the migrated keys and cannot even see the unmigrated one.
3. **The runbook still described the old world.** The setup doc for the new credential said, in a fenced code block ready to paste: append to `.env`. The instruction predated the migration and nobody swept docs for writers when the file changed owners. This is why it happened twice: the error was not a person's habit, it was *installed* in the procedure. A doc is a writer too.

## The Pattern

When a hand-maintained file becomes a generated one, the migration is not done when the renderer works. It is done when **every writer that targeted the old file has been found and repointed** — human habits, runbooks, scripts, and agents alike.

1. **Grep for writers, not just readers.** The migration checklist naturally covers consumers ("does everything still read the right values?"). The killers are the *producers*: `>> path`, `sed -i`, editor instructions in docs, `echo ... >`. Sweep the repo and the runbooks for the old file's path the day the renderer lands.
2. **Put the DO-NOT-EDIT header in the artifact itself,** naming the template, the renderer, the trigger, and the consequence ("any key not in the template is destroyed on the next restart"). The person about to append is looking at exactly one file; that file is the only place a warning is guaranteed to be seen.
3. **Fix the runbook in the same change as the renderer.** An instruction that says "append to the artifact" is a standing order to reintroduce the bug. If the doc can't safely quote the new syntax (this template language parsed its own reference syntax even inside comments — including inside *documentation* rendered through it), point at the live template instead of inlining an example.
4. **Give the gate the whole file, not just the template's half.** A sanity check that only validates known keys silently blesses the deletion of unknown ones. Cheapest version: compare rendered key-set against the previous artifact's key-set and warn on any key that would vanish — that one check converts "silently scheduled for deletion" into a named diff at render time.
5. **The artifact's failure consumer needs a reader.** The credential's consumer refused loudly and correctly — into a cron-redirected log that no monitor tailed. Detection existed at every layer and terminated in a file nobody read (see: instrumentation-with-no-reader). Wire the artifact's consumers' fatal path into a surface that reaches a human.

## Why It Works

The writers are the whole failure surface. The renderer, the template, and the vault can all be correct forever; one stale `>>` in a runbook re-creates the incident on schedule. Sweeping writers once, at migration time, is cheap because you know exactly what changed; discovering them one outage at a time is expensive because each loss looks like a fresh mystery — the file *had* the key, and now it doesn't, and the process that removed it ran at an unrelated moment with no log line pointing back.

## Trade-offs and Limits

- **The header is advice, not enforcement.** Anything that regenerates or truncates the artifact also deletes the warning. Enforcement lives in the render-time key-set diff (step 4), the header just prevents honest mistakes.
- **Emergency writes to the artifact are still legitimate** — a render pipeline that is itself broken must not lock you out of the runtime file. The header should say how to make the fix durable afterward, not pretend the file is untouchable.
- **Related but distinct:** lawful-write-surface is about an agent with *no* legal place to write; this is about a place that *stopped* being legal and kept accepting writes. migration-blinds-readers is the read-side twin — a moved store blinds consumers; here the store stayed put and the *authority over its contents* moved.

## Recurrence — 2026-08-23, through an alias the header cannot reach

Six days after this pattern was written, the same file ate another credential — this time appended by a Claude session that had *read this pattern's source incident in the project docs*. The writer never saw the DO-NOT-EDIT header because it never believed it was in this file: the write went to `~/.openclaw/workspace/.env`, a **symlink** into the artifact. Every probe reinforced the wrong identity — `stat` on the link path reported the symlink inode's own `777` and its April creation date, which read as "some old workspace env file", not "the render target". The appended VPN credentials verified, interpolated, and worked; the next deploy's gateway restart re-rendered the artifact and recreated the VPN container with blank credentials mid-deploy.

Two additions the first writeup missed:

1. **The writer sweep (step 1) must include aliases.** `grep` for the artifact's path finds writers that name it; it cannot find writers that name a symlink, bind mount, or docker volume that resolves to it. Sweep with `find -lname '*<artifact>*'` (and the mount table) the day the renderer lands, and name the aliases in the header itself — the alias is a door into the file that carries none of the file's warnings.
2. **Step 4 is now mechanized here, and the control was run the same day.** The renderer diffs the live artifact's key-set against the render before the `mv`; a hand-added key about to vanish is named in the journal and appended to a breadcrumb log that the fleet's session-start snapshot surfaces with a ⚠ for 7 days. Positive control: a dummy key appended and rendered away announced itself by name at both surfaces. The fix for the lost credential itself moved it out of the env lane entirely — the consumer (gluetun) reads its own secret files from a directory nothing regenerates, which is the "consumer's own store" escape this pattern's remedies kept circling without naming: **when a credential has exactly one consumer, the render pipeline is accidental complexity — put the secret where the consumer looks and no renderer can ever eat it.**
