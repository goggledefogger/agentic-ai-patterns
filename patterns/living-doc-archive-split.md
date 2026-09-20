---
type: pattern
date: "2026-06-01"
source: The work vault — a doc de-bloat (the file that /brief and /today auto-loaded every session)
tags:
  - documentation
  - context-cost
  - context-engineering
  - living-doc
  - pruning
---

# Archive-Split for Auto-Loaded Living Docs

A living doc that gets auto-loaded every session (a TODO, a status snapshot, anything a `/brief` or `/today` routine reads) is a recurring token tax. Left alone it accretes closed items and historical context until most of what loads into every session is dead weight. The fix is to split closed/historical content out to a sibling file that is *not* auto-loaded, wire the prune into an existing ritual so it stays split, and — the part that bites — never let the prune touch open items mechanically.

This is the bloat problem. It is *not* the drift problem ([`living-doc-refresh-ritual.md`](living-doc-refresh-ritual.md)). Drift is "the doc says something untrue now." Bloat is "the doc is true but three times longer than it needs to be, and you pay for it every session." Same docs, different failure, complementary fixes.

## The Problem

Three forces compound:

- **Auto-load makes the cost invisible-but-real.** If a session-start routine reads the file, every session pays its full token count before doing any work. Nobody sees a bill, so nobody prunes.
- **Append is easier than curate.** Every meeting, every closed thread, every resolved ticket adds lines. Removing them is a decision; adding them is a reflex. The file only grows.
- **Closed ≠ removed.** People check items off (`[x]`, strikethrough) and leave them in place as a record. The record is worth keeping — just not in the file that loads every session.

The employer instance: a `TODO.md` reached ~1,300 lines / ~57K tokens, read in full by two daily routines. Roughly 230 of those lines were closed or struck items still riding into every session, plus whole historical sections months stale. After the safety checks filtered out items that looked closed but weren't, 205 actually moved.

## The Pattern

### 1. Split closed/historical out to a non-auto-loaded archive

Create a sibling file (`TODO-archive.md`, `status-archive.md`). Move closed `[x]`, struck-through, and verified-historical whole sections into it. **Move, not delete** — every line lands verbatim in the archive, greppable forever. The archive is *not* embedded in any dashboard and *not* read by the session-start routines. It exists for `grep`, not for context.

### 2. Wire the prune into an existing ritual

A one-time split decays back to bloat unless something keeps it split. Don't invent a new mechanism. Extend a forcing function you already run (see [`wire-into-existing-flows.md`](wire-into-existing-flows.md)):

- **Active ritual runs the mutation.** The end-of-day or wrap routine moves completed items to the archive. Do not auto-run this mutation in a git pre-commit hook because mutating files mid-commit is error-prone and bypasses the human safety pass for open items
- **Passive git hook acts as the guard.** Write a git pre-commit hook that counts closed items or line numbers. The hook blocks commits and warns the developer if the file exceeds its size limit (like ~450 lines or 25 closed items), forcing them to run the pruning script explicitly


### 3. Pruning safety: mechanical on closed, human on open

This is the rule that took a near-miss to learn: **only mechanically archive closed `[x]` / struck items and whole sections you have verified are pure historical record. Never auto-archive an open `[ ]` item by date or keyword.**

- **"Past-dated ≠ resolved."** An open item under a March heading is not automatically dead — it may be live work filed in an old section. A "verified-dead" label applied by date inference is a lie waiting to happen.
- **Live-vs-dead is irreducibly semantic.** A narrow matcher (keyword/date) buries live items; a broad safety net rescues everything and prunes nothing. There is no regex that separates "stale-dead" from "stale-but-live" when both mention the same active project. Only judgment does.
- So run it as **two passes**: Pass 1 mechanical (closed/struck/verified-historical — non-regressing by construction), Pass 2 human-gated (the stale-open items, confirmed one by one or left alone).

### 4. Verify with invariants before you write

A bulk move is a bulk-corruption risk. Gate it on a dry-run that checks two invariants and refuses to write if either fails:

- **Conservation:** every line that leaves the source file appears verbatim in the archive. (A reworded survivor shows up as a remove+add pair — catch it.)
- **Live-anchor survival:** every active ticket / thread / id still resolves in the source file after the move.

The harness that checks these *before* writing is what catches the regression while it's still cheap. In the employer run, two dry-runs each tried to bury a live item (a candidate redirect filed under a closed thread) and the live-anchor check stopped both.

## The Floor: the bloat is often live, not archival

After the mechanical pass, you hit a floor. The employer `TODO.md` dropped ~57K → ~47K and *stayed* large — because the remaining bulk was open, live 1:1 follow-ups, not dead checkboxes. **Mechanical pruning has a safe floor; below it the file is big because the work is big.** Going lower is then one of two different projects, and naming which one matters:

- **Judgment pruning** — adjudicate the stale-open items (slow, human, per-item).
- **Structural change** — the content shouldn't live here at all (e.g. per-person follow-ups belong in the person's file, surfaced on demand, not piled into one global list).

Don't mistake the floor for a failure of the prune. The prune did its job; what's left is a content-architecture question.

## Example

The work vault, 2026-06-01:

- `TODO.md` ~57K → ~47K tokens, `status.md` ~14K → ~9K tokens. ~15K fewer tokens loaded by `/brief` and `/today` every session.
- 205 lines moved to `TODO-archive.md`, conservation-checked (all 205 verbatim, zero dropped).
- The prune wired into the wrap routine (`/close-day` moves on close, `/ready-to-wrap` prunes on a size trigger).
- An independent fresh-session audit (clean context, cross-checking the archive against the live status doc) caught the one defect the author missed: a subsection labeled "verified-dead" that still had a live close-the-loop thread. Nothing was lost — it survived in the source file — but the *label* overclaimed. (See [`ask-challenge-verify-correct.md`](ask-challenge-verify-correct.md) / [`contradiction-surfacing.md`](contradiction-surfacing.md) for why the independent reader catches what the author can't.)

## When to Use

- Any living doc an automated routine loads every session (the token cost is the trigger).
- Any append-mostly doc where closed items accumulate (TODOs, status snapshots, changelogs-in-prose).

## When NOT to Use

- Docs read by a human on demand, not auto-loaded — the token cost isn't being paid repeatedly, so the archive overhead isn't worth it.
- Small docs that haven't crossed the bloat threshold. Don't pre-split a 100-line TODO.
- Docs whose bulk is *live* content — that's the floor case above; archiving won't help, you need judgment or a structural change.

## Adjacent Patterns

- **[`living-doc-refresh-ritual.md`](living-doc-refresh-ritual.md)** — the complementary failure. Drift (untrue) vs bloat (too long). The refresh ritual keeps the living layer *honest*; this keeps it *lean*. A doc needs both.
- **[`wire-into-existing-flows.md`](wire-into-existing-flows.md)** — the prune must wire into a ritual you already run, or the split decays back to bloat. The single most important step.
- **[`doc-warning-preamble.md`](doc-warning-preamble.md)** — pairs well: the preamble tells a reader the doc may be stale; the archive-split keeps the doc short enough to be worth reading.
- **[`smoke-tests-with-real-data.md`](smoke-tests-with-real-data.md)** — the dry-run conservation/live-anchor harness is this pattern applied to a bulk edit: verify against the real file, not a mock, before writing.
- **[`contradiction-surfacing.md`](contradiction-surfacing.md)** / **[`ask-challenge-verify-correct.md`](ask-challenge-verify-correct.md)** — the independent fresh-session audit that catches the author's own over-broad "dead" labels.
