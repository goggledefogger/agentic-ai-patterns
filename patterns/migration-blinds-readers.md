---
type: pattern
date: "2026-06-30"
source: The household-agent repo (a household agent on a Raspberry Pi), Hermes 2026.6.x storage migration
tags:
  - upgrades
  - diagnostics
  - drift
  - verification
---

# Storage Migration Blinds Your Readers

When a dependency migrates the store it owns to a new format, every tool that reads the old format keeps running and returns empty. It does not crash. It reports "nothing here," and you trust the emptiness.

## The Pattern

A platform upgrade moves session/event/state storage to a new backend (per-file JSONL to a single SQLite database, flat logs to a table, v1 schema to v2). Your readers were written against the old shape. After the upgrade they still run, still exit 0, and still print a result. The result is just empty, because the old files stopped growing and the new store is invisible to them.

The trap is that "the store has no rows" and "I cannot read this store" produce the identical output: nothing. A reader that finds nothing has to prove it can still see the store, or its silence is meaningless.

Concrete instance: Hermes 2026.6.x moved session storage from `~/.hermes/sessions/*.jsonl` to a single `~/.hermes/state.db`. The catch-up reader globbed `*.jsonl`. After the upgrade it found the last frozen pre-upgrade file and reported "No session files found" for everything since. A real voice-initiated conversation and a failing cron job were both invisible. The catch-up confidently summarized "no conversations in 24h," and the agent reading it repeated that as fact. The data was there the whole time, in a file the reader never opened.

## Why It Works

The fix is three habits after any storage migration:

- Audit every consumer that opens the store. The migration note that mentions a schema bump is the cue, grep for every tool that touches the old path
- A reader that returns empty should distinguish "store has no rows" from "I cannot parse this store." Probe the new format, do not just glob the old one and shrug
- Treat a sudden "everything went quiet" right after an upgrade as suspect, not as good news. Quiet is the symptom this failure produces

When the canonical store carries the full history (migrated-in old rows plus new ones), read it as the source of truth and dedup the frozen old fragments. The old files are not deleted, they are just frozen, so a same-id fragment is redundant, not extra signal.

## When to Use

Any time you upgrade a dependency that owns your session, event, or state storage. The release notes will mention a schema or storage change. That line is your trigger to find every diagnostic, dashboard, cron, and catch-up step that reads the affected store, and confirm each one can see the new format before you trust a single "all quiet" report from it.

## Adjacent Patterns

- `self-reporting-staleness-check.md` finds drift in your corpus. This finds the case where the tool that would report drift has itself gone blind
- `stale-pointer-asserts-confidently.md` is this hazard's other half. Here a migration makes a reader go *quiet*; there it makes a copied fact in another repo answer *confidently and wrongly* — and the migration's own cleanup pass cannot reach across the repo boundary to fix it
- `system-understanding-protocol.md` is the discipline that should have caught the false "no conversations" claim. A blind reader is exactly the partial state you must not report from

## Source

The household-agent repo `read_messages.py`, blind to `state.db` after the 2026-06-28 Hermes 2026.6.x upgrade moved storage off `*.jsonl`. The reader was given a SQLite source so the display loop is format-agnostic, with same-id JSONL fragments deduped against the canonical store.
