---
type: pattern
date: "2026-08-10"
source: The household-agent repo — adding a mid-flight download size guard; the injected-downloader test that proved it found that every abort path had been killing nothing for weeks, guarded by a grep whose comment claimed "verified empirically"
tags:
  - testing
  - verification
  - anti-pattern
  - debugging
---

# A Source Assertion Pins Your Belief, Not the Behavior

A test that greps the implementation can only tell you the code still *says* what you decided it should say. When the thing under guard is a runtime relationship — which process is whose parent, what a lookup returns, whether a signal lands — the spelling and the behavior are independent, and a source assertion guards the wrong one. It will hold the line perfectly while the behavior is absent.

## The incident

A download wrapper aborted on stalls, on decoy sources, on garbage content, and on explicit cancel. All four paths ended in `_kill_download_children "$$"`.

A prior session had been bitten by this exact call using the wrong pid, fixed it, and left a regression guard:

```bash
if grep -Eq '_kill_download_children[[:space:]]+"\$PPID"' "$_dm"; then
    printf '  FAIL  kills the downloader via $PPID (orphans yt-dlp)\n'
else
    printf '  ok    no $PPID — aborts target $$ (the yt-dlp parent)\n'
fi
```

with a comment above it reading *"PID SEMANTICS (verified empirically)"*.

It was not verified empirically. It was reasoned, convincingly, and the reasoning was wrong: bash freezes `$$` and `$PPID` at shell startup and never updates either inside a subshell, so inside the tracker function `$$` is the downloader's **grand**parent. `pgrep -P "$$"` matched nothing. Every abort wrote its status file, logged its verdict, sent its Telegram — and killed nothing at all.

The guard was green the entire time, because the source did say `"$$"`. It was pinning a belief.

It surfaced only when a new feature needed the kill to actually work, and the test written for that feature injected a fake downloader and asserted on the *process*: is it dead? Measured pids: main `823560`, background subshell `823561`, downloader ppid **`823561`**. Neither `$$` nor `$PPID`.

That same test then caught two more, both invisible to any source scan: passing `"$BASHPID"` inline to a backgrounded command expands it *in the forked child*, so the correct-looking fix handed the function its own pid; and purging files inside the tracker raced the reap that its own kill triggered.

## The Pattern

Ask what kind of fact the guard is protecting.

- **A convention** — naming, layering, an import that must not appear, a branch that must record a skip. A source assertion is genuinely the right tool. Wiring is a textual fact.
- **A runtime relationship** — process ancestry, signal delivery, what a query returns, whether a timer fired. Only a runtime probe can observe it. Grep can confirm the call is present; it cannot confirm the call *did* anything.

The tell is the verb in your test's description. "…targets `$$`" is a claim about text. "…kills the downloader" is a claim about the world. If the description promises the second and the assertion inspects the first, the gap is where the bug lives.

Cheap runtime probes are usually available and usually skipped as too much trouble:

```bash
# Inject a fake whose only job is to be observable, then assert on the world.
printf '#!/usr/bin/env bash\nwhile true; do sleep 1; done\n' > "$TMP/bin/yt-dlp"
# ... run the real code path ...
assert_eq "" "$(pgrep -f "$TMP/bin/yt-dlp")" "downloader killed, no orphan left"
```

That is nine lines. It found three defects that four source-asserting tests and two careful code comments had not.

## The louder half: "verified empirically" is a claim, not a measurement

The comment is what made this survive. A future reader — including the author, months later — sees *verified empirically* and correctly declines to re-probe something already checked. The phrase converts an unverified belief into a load-bearing one, and every subsequent reader inherits it.

So treat verification language as an assertion that itself needs backing:

- **Record the measurement, not the verdict.** Not "verified empirically" but "measured 2026-08-10: main 823560, subshell 823561, child ppid 823561". A reader can check numbers. Nobody can check an adjective.
- **If you cannot produce the numbers, do not claim the verification.** "Reasoned, not probed" is a completely respectable comment and it invites exactly the re-check that "verified" forecloses.
- **When you do re-probe, watch it fail first.** The new test above was run against an unfixed copy and went red, naming the exact line. A guard nobody has seen fail is a guard nobody has tested.

## Related

- [[green-tests-can-mirror-the-same-guess]] — the sibling failure: tests written by the same agent as the code confirm the code's inventions. Here the test and the code disagreed with *reality* in the same direction, which is how both stayed green.
- [[tests-pin-substance-not-identifiers]] — same shape one level down: an assertion keyed on a name passes on anything that merely mentions the name.
- [[a-partial-read-proves-presence-not-absence]] — a grep finding the string proves the string is there; it never proved the behavior was.
