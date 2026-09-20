---
type: pattern
date: "2026-07-31"
source: The agent's own repo, 2026-07-31. One morning's cron-failure alert carried two backwards verdicts — a PASSING smoke test flagged ERROR because its healthy JSON tail ("ok":true, "failures":[]) matched \bfailures?\b on the KEY NAME, while the genuinely failed download on the same page classified "unknown" because "failed" was in neither pattern list. A one-hour survey then found 15 sites of the same class across the repo, including a duplicate gate that had refused every cron-add since the day it was written ("NOT FOUND" contains "FOUND").
tags:
  - verification
  - detectors
  - shell
  - monitoring
  - alerting
---

# Verdicts From Structured Signals, Not Prose Matching

A health check classified cron jobs by regexing the last line of each log. One morning its alert carried two verdicts that were exactly backwards: a **passing** smoke test read as ERROR (its healthy JSON tail `"ok": true, "failures": []` matched `\bfailures?\b` on the key name alone), and a **genuinely failed** download read as merely "unknown" (`failed` was in neither the error list nor the success list). The classifier wasn't under-tuned — it was reading the wrong layer.

This is the runtime sibling of `grammar-parsing-over-text-matching.md`. That pattern is about detectors reading *source code*; this one is about verdicts derived from *runtime output* — logs, API responses, command output — where the fix is not an AST but a signal hierarchy.

## The Pattern

Whenever code decides something semantic — success/failure, healthy/broken, present/absent — by substring or regex over free-form text, it inherits every future wording change, spacing variation, and coincidental containment as silent verdict flips. A one-hour survey of one repo found fifteen live sites. The recurring shapes:

1. **Containment traps.** A duplicate gate piped a checker's output through `grep -q "FOUND"` — but the miss message was `NOT FOUND: ...`, which contains `FOUND`. Every add refused as a duplicate *since the day the gate was written*. (The checker returned a correct exit code the whole time; the pipe discarded it.)
2. **Structured data regexed instead of parsed.** `grep '"ok":true'` on an API response: a real success with a space after the colon reads as failure; error prose containing the literal text reads as success. This was the *last hop of every alert in the repo* — a wrong verdict there is a silently swallowed notification.
3. **Prose-status passing sick states.** `docker compose ps` piped to `grep -qi "up\|running"`: a container in `Up 3 hours (unhealthy)` passes the very check that exists to catch it. The structured `{{.State}}`/`{{.Health}}` fields were one flag away.
4. **Fail-open gates.** A diagnostic greps remote output only for its alarm token (`STUCK`, `WARN`, `BLOATED`). Every unanticipated output — a parse failure, a missing file, a shape the author never saw — matches nothing and reads as *healthy*. The most dangerous verdict a diagnostic can emit is a silent pass over output it didn't recognize.
5. **Free-text triggers with side effects.** "Restart the production container if the accumulated failure prose contains `hub`" — fired by URLs, container names, and its own recovery notes; missed a hub-served page whose name didn't happen to contain the word.
6. **Tool-prose keyword lists that only grow.** A quarantine verdict keyed on `file -b` output matching `video|MPEG|Matroska|...` — a list that had already grown three entries chasing false-positive quarantines of real files, when `ffprobe -select_streams V` answers the actual question ("is there a playable video stream?") in one structured call.
7. **A counter read as a verdict.** A summary line reports `done — scanned=5 optimized=0 skipped=5 failed=0`. That is a *clean* run stating its tallies, but `\bfailed\b` matches and the job is alerted as ERROR every week. The classifier had already been hardened against the prefix form — lookbehinds excused `0 failed` and `no errors` — and the suffix form walked straight past them, because the guard was written against the phrasing someone imagined rather than the phrasing the job emits. **A tally is not an outcome.** The words inside it name what was counted, not what happened.

### The counter case deserves its own warning, because fixing it can invert it

The obvious repair is to teach the classifier that `failed=0` is benign. Do that carelessly and it will suppress real failures instead of inventing fake ones.

In the same file, the transient-skip short-circuit was `\bskip\b`, which does not match `skipped` (no word boundary before the doubled `p`), so a job logging `skipped — under the rotation threshold` was reported "unclassifiable" every week. Widening it to `\bskip(?:s|ped|ping)?\b` fixed that job — and simultaneously made the summary line above short-circuit to **OK on the word `skipped=5`**, before the error scan ever ran. A genuine `failed=2` on that same line would have gone silent. One edit turned a false alarm into a false all-clear, in the direction nobody watches.

That inversion was caught by a **positive control** asserting `failed=2` still fires — a test written alongside the fix specifically to prove the fix hadn't gone too far. Without it, the change would have shipped looking strictly better than what it replaced.

So: strip counters *before* any verdict logic reads the line, and keep the two directions honest against each other.

```python
# A tally is not a verdict. Neutralize counters before classifying,
# but let a NON-zero failure count survive as a failure.
line = re.sub(r'\b(?:fail(?:ed|ures?|s)?|errors?)\s*[=:]\s*0+\b', ' ', line, flags=re.I)
line = re.sub(r'\b(?:skip(?:s|ped|ping)?|scanned|optimized)\s*[=:]\s*\S+', ' ', line, flags=re.I)
```

Whenever you loosen a detector to stop a false alarm, add the test that proves it still fires on the real thing. A detector's two failure directions are not symmetric in visibility: false alarms reach you every week, silent passes reach you never.

## The Fix: A Signal Hierarchy

Take the strongest signal available, in this order, and only fall through when the level above genuinely doesn't exist:

1. **Exit codes.** The process already voted. A pipe (`cmd | grep ...`) silently discards the vote — the containment trap above was a correct exit code thrown away.
2. **Machine-readable output modes.** `--format json`, `--format '{{.State}}'`, `systemctl is-active`, `ffprobe -select_streams V`, `ip -j`. If the line is JSON, `json.loads` it and read the field — never regex a serialization.
3. **Deliberate sentinels.** When you control the writer, emit a line *designed* to be parsed: `STATUS: OK`, a `__SENTINEL__` marker, one enum per line (`THINKING_ONLY|session|model`). A sentinel is a contract; log prose is not.
4. **Keyword heuristics, flagged as such.** For genuinely unstructured third-party text, keep the regex — but its verdict is "looks like X, low confidence," never a silent pass. Route unmatched output to UNKNOWN/inconclusive, not to OK.

And at every level: **enumerate the pass states and fail closed on anything else.** `grep -q ALARM || healthy` is fail-open. The gate should be: alarm states alarm, *recognized* pass states pass, everything else surfaces as "unrecognized output — NOT a pass."

## A Structured Signal Can Still Be the Wrong Layer, If It Is Coarser Than the Question

The hierarchy above reads as structured-vs-prose, and the sites it found were all prose. There is a second version that evades that framing entirely: the code *is* reading a structured signal, correctly, and that signal is simply **coarser than the distinction being drawn**. Coarseness collapses distinct conditions into one verdict, and no amount of parsing rigour recovers them, because the information was never in the field being read.

A model-liveness probe in the household-agent repo classified on HTTP status:

```bash
if   [[ "$code" == "200" ]]; then  echo "  ✓ $model"
elif [[ "$code" == "404" ]]; then  echo "  ✗ $model RETIRED"
else echo "  ⚠ $model probe returned HTTP $code (transient? key issue?)"
fi
```

A status code is genuinely structured — an integer with defined semantics, not prose — and this is exactly the shape the pattern above recommends. It was still the wrong layer, because HTTP 429 is at least two conditions: a per-minute rate limit that clears itself, and a hard billing stop that clears only when a human pays. The probe rendered a total service stop as `(transient? key issue?)` — neither, and the response body sitting in the temp file it had already written said so in plain English:

```json
{"error":{"code":429,"message":"Your prepayment credits are depleted.",
          "status":"RESOURCE_EXHAUSTED"}}
```

The fix is the same move one level finer: classify on the body, keep the code as the outer branch, and preserve the distinction the coarse field had flattened — `BILLING` (loud, actionable, not self-clearing), `QUOTA` (transient, with a "if it persists this isn't a blip" caveat), `UNKNOWN` (quotes the body rather than guessing).

Two things this case adds to the discipline:

- **`UNKNOWN` must still quote the raw signal.** The point of falling through is that you didn't recognise it; discarding it leaves the next reader with less than the program had. Fail-closed and *hand over the evidence*.
- **The negative control is the finer verdict's own opposite.** Pinning "prepay-depleted classifies BILLING" is half a test. Without a real rate-limit body pinned to stay `QUOTA`, the natural next change is to escalate all 429s — which destroys the distinction just built, in the other direction. Capture both bodies from production and pin both.

The general tell: a verdict field with fewer possible values than the conditions you need to act on differently. Ask what the caller will *do* with each verdict; if two verdicts demand different actions and share a value, you are reading too coarse a layer — even if that layer is impeccably structured.

## Second-Order Lesson: Backwards Verdicts Are One Bug, Not Two

The same classifier over-fired (healthy JSON → ERROR) and under-fired (real failure → unknown) on the same morning's report. As with source-code detectors, the tell of reading the wrong layer is *simultaneous* false positives and false negatives — tuning the pattern list fixes one direction at a time, forever. Changing what the code reads (parse the JSON; add the structured branch first) fixed both in one edit.

## Third-Order Lesson: Survey the Class, Fix by Blast Radius, Say the Stopping Rule

Once one instance is found, the class is cheap to enumerate: one focused agent pass over the repo (grep decision points — `grep -q` gating an if, regex over subprocess output, JSON substring checks) produced a ranked 15-site list in minutes. Then *don't* fix all fifteen. Rank by blast radius (the alert last-hop and the always-refusing gate outrank a display-only label), fix the top tier, and record the rest with an explicit stopping rule — otherwise the sweep becomes an infinite churn generator, which is its own anti-pattern.

## Why It Works

- Exit codes and JSON fields are contracts; log wording is not. Verdicts built on contracts survive reword, respace, and reorder.
- Fail-closed enumeration converts "the author didn't anticipate this output" from a silent pass into a visible finding — the difference between a diagnostic and a placebo.
- The hierarchy keeps the fix proportionate: you rarely need a parser; usually the structured signal already exists and is being actively discarded (a piped-away exit code, an unparsed JSON line, an unused `--format` flag).
- Regression tests become trivial and permanent: the literal log lines that produced the backwards verdicts go in as fixtures.

## When to Use

- Any `grep`/`re.search` whose match changes a verdict, alert, state file, or triggers a side effect (restart, quarantine, notification)
- Anything consuming another tool's output when that tool offers `--format`, `--json`, or a meaningful exit code
- Health checks and diagnostics — audit specifically for fail-open gates: "what does this report when the output is something I never imagined?"
- The last hop of an alerting path, where a wrong verdict is invisible by construction

## When NOT to Use

- Display-only matching (colorizing a log tail, choosing an emoji) — a wrong match costs nothing; don't ceremonialize it
- Genuinely unstructured third-party text with no structured alternative — keep the heuristic, but demote its verdict to low-confidence and fail closed around it
- Don't build a parser when a sentinel will do: if you own the writer, emitting `STATUS: OK` is ten minutes; parsing its prose is a treadmill

## Adjacent Patterns

- `grammar-parsing-over-text-matching.md` — the same root cause one layer down: detectors reading source *code* as characters. Together they cover both places text-matching verdicts hide (reading code, reading output)
- `unrun-checks-read-as-passing.md` — the fail-open gate is the runtime version: a check that didn't recognize its input reporting the same "all clear" as a check that passed
- `verification-needs-a-negative-control.md` — after restructuring a classifier, prove both directions with the literal lines that were misjudged
- `decorative-gate.md` — the containment-trap gate (refuses everything / passes everything) is a decorative gate created by accident

## Source

The agent's own repo, 2026-07-31. Entry point: one morning's cron-health Telegram alert with two backwards verdicts (`check-cron-health.sh` — passing streaming-smoke JSON flagged ERROR via the `"failures"` key name; failed download flagged unknown because `failed` matched no pattern). The survey that followed found 15 sites; the top six fixed the same day: the cron-health classifier (JSON parsed before text heuristics), an automation-registry duplicate gate that had refused every cron-add since creation (`NOT FOUND` contains `FOUND`; exit code was being discarded by a pipe), the Telegram send verdict (last hop of every alert, substring → parsed JSON), a compose post-check that passed `unhealthy` containers (Status prose → State/Health fields), a hub smoke test whose auto-restart fired on free-text containment (→ explicit failed-check-name membership), and a media-download quarantine keyed on `file -b` wording (→ `ffprobe -select_streams V`). Four fail-open diagnostic gates also converted to enumerated-pass fail-closed. Each fix carries a regression test using the literal misjudged log line as its fixture.
