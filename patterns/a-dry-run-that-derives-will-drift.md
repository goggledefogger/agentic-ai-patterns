---
type: pattern
date: "2026-08-05"
source: The personal agent session 2026-08-05 — a promotion script's --dry-run reported byte counts the real write never produced, twice in one hour, from two unrelated-looking bugs with one root cause. Both were caught by reading the preview against the actual result, not by tests.
tags:
  - verification
  - dry-run
  - human-in-the-loop
  - agent-safety
  - drift
---

# A dry run that derives will drift

A dry run is not a convenience feature. It is a **consent affordance**: the thing a human reads in order to authorize an action they cannot undo. Whatever it prints is trusted by construction, because the whole point is that nobody has checked the real thing yet.

That makes it the one output in a tool where being *slightly* wrong is worse than being absent. A missing preview makes a person cautious. A confident wrong preview makes them approve.

There are two ways to build one, and only one of them survives:

- **Echo** — the preview renders the exact artifact the action will consume. The message that will be sent, the command that will run, the bytes that will be appended.
- **Derive** — the preview computes a *property* of the action: a size, a count, a cost, a duration, a row total.

Echo cannot drift, because there is only one object. Derive is a second expression of the same intent, living in a different branch from the code that acts, and it will diverge. Not might.

## The incident

A script promoted staged content into protected files: split a staged note on its headings, append each block to its named target, delete the staging file. `--dry-run` printed projected byte counts so a human could sanity-check before authorizing the write.

The preview computed `adds = len(body) + 2`. The writer emitted `fh.write("\n\n" + body + "\n")`. Two expressions of one intent, and two bugs fell out of the gap inside an hour:

1. **No running state.** Two blocks named the same target, and each projection started from the file's on-disk size — describing a world where every block is the first one. The dry run promised `3045 -> 4460` for a file already headed to 6747.
2. **Characters, not bytes.** `len()` counts characters; the file receives UTF-8. The staged blocks were full of em dashes at 3 bytes each, so the projection under-reported by 15 and 3 bytes — numbers a human had already been quoted.

Neither bug was in the writing. The appends were correct every time. Only the *promise* was wrong, which is the half a person reads before saying yes.

The fix collapsed the two paths into one: the planner builds the payload string, the writer writes **that same object**, and the size is measured on it encoded. Nothing to keep in sync, because there is no longer a second thing.

## The Pattern

1. **Build the artifact once; let the preview render it and the action consume it.** If the dry-run branch and the real branch both construct something, that is the bug, whatever the current numbers say.

2. **If a derived number must be shown, derive it from that shared artifact, serialized the way the action will emit it.** `len(payload.encode())`, not `len(body) + 2`. The encoding step is where character counts quietly become lies.

3. **Assert the projection against the real delta in a test — with a case that would expose encoding.** An ASCII-only fixture passes forever and proves nothing. Put a multi-byte character in the fixture body, apply for real, and assert the predicted size equals the size on disk.

4. **Carry running state across items.** Any preview reporting per-item outcomes must thread the accumulated result forward, or it silently describes N independent first-writes.

## Why it outranks an ordinary display bug

Normal display bugs fail toward confusion, and someone squints and asks. A wrong dry run fails toward **confident approval** — it produces exactly the plausible-looking number that gets waved through. It also breaks the mechanism it is embedded in: patterns that resolve a permission deadlock with "deterministic script, human triggers it after reviewing `--dry-run`" are only as sound as the preview's truthfulness. Corrupt that and the human-in-the-loop is a rubber stamp holding a fabricated summary.

"It's only the display" is the sentence to catch. For a preview tool, the display *is* the product.

## When NOT to use

When the effect genuinely cannot be known before running — an external system assigns the id, the remote decides the byte count, the API sets the price — do not manufacture a projection. Print `unknown` and say which part is unknowable. A preview that guesses is this same failure with extra confidence.

## Watch-outs

- **Echo is the safe default; reach for derived numbers only when the payload is too large to show.** A preview that prints the literal message to be sent has no drift surface at all.
- **Sharing a helper is not sharing an object.** Two callers of the same function still diverge the moment one adds a wrapper, a newline, or an encoding step the other lacks.
- **The bug hides from tests and appears on first real use.** Both instances here were caught by comparing the preview to the actual result on a live run, because the tests asserted behavior of each path separately and each path was internally consistent.
- **Audit sibling tools by shape, not by symptom.** A repo scan for dry-run flags separates cleanly into "prints the payload" (safe) and "prints a number about the payload" (suspect). Only the second class needs the fix.

## Adjacent Patterns

- `phase-is-computed-ui-is-rendered` — the general form: one function computes, every consumer wraps that same machine, never a second derivation
- `escape-hatch-is-the-denied-tool` — the deadlock resolution whose safety rests on this preview being true
- `decorative-gate`, `unrun-checks-read-as-passing` — controls that report a verdict they did not earn
- `opaque-write-needs-a-read-back` — check the artifact, not the tool's claim about it
- `verification-needs-a-negative-control` — a test that cannot fail proves nothing, which is what an ASCII-only preview fixture is

## Source

The personal agent session 2026-08-05, a staged-content promotion script. Two bugs, one hour, one root cause; both surfaced by reading the preview against the real result rather than by any test.

## Also seen: 2026-09-09, the derive-twice problem across TIME rather than across branches

Same root, different axis. Instead of a preview branch and a write branch deriving the same value, an agent derived the *same fact* half a dozen times over two days with successive throwaway scripts — and the answers moved: 35,334 then 63,497 distinct files; 2,189 then 2,947 album-only media; byte totals returning 0.0 GiB from a misread Zip64 field. The archive never changed; each ad-hoc pass brought its own filters and its own idea of what counted as the same item. The user noticed before the tooling did: *"you keep realizing things you thought were different before."* Fix is this pattern's — build the artifact **once**: one index, reconciled against the source's own totals and refusing to be trusted if any disagree, with the normalisation rule in exactly one function, and every later count a query against those rows.
