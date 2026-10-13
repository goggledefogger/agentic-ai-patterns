---
type: pattern
date: "2026-10-07"
source: A helper driving a headless browser reported that a feature never appeared, and the cause was a value it had written in the wrong format
tags:
  - agents
  - subagents
  - debugging
  - evidence
  - anti-pattern
---

# A Helper's Negative Result Reports Its Setup

When a helper agent comes back with "it never showed" or "nothing found", the result describes the helper's setup before it describes the system. Until you check that the setup has the shape the code reads, the negative is about the test, not the feature. Before believing a negative, read the reader.

## The Problem

A web app shows a "new since your last visit" pill when items have been added after the visitor's previous visit. A helper driving a headless browser was asked to confirm it. It set the last-visit value in local storage to a date in January, reloaded, and reported back: the pill never appeared, even with the date set months ago.

That reads like a bug. It was not one.

A second helper read the code. The app reads the value with `Number(localStorage.getItem(key))` and expects epoch milliseconds. The first helper had written an ISO date string. `Number("2026-01-01")` is `NaN`. The code treats `NaN` as no previous visit, which counts as a first visit, and on a first visit nothing is new. So 0 new items, no pill. The feature was fine. The test had fed it a value it could not read.

## The Pattern

1. **Treat a helper's negative as a claim about its setup.** "It never appeared" means "under the conditions I created, it never appeared". The conditions are part of the result
2. **Read the reader before the result.** Find the code that consumes whatever the helper set up: the storage key, the config field, the query parameter. Check the type, the format and the units it expects
3. **Compare shapes, not intentions.** The helper meant "last visit in January", and so did the code. The string and the number disagreed anyway
4. **Ask helpers to report the exact value they wrote.** "Set last visit to January" hides the format. `"2026-01-01"` shows it
5. **Make a negative earn its place.** Before a negative turns into a bug report, there should be one run where the same setup produced a positive, or a reading of the consuming code that rules the setup out

## Why It Works

Positive results have a built-in check: something happened, and you can look at it. Negative results do not. A wrong setup and a broken feature look exactly the same from outside. Helpers make it worse because they summarize. The summary keeps the conclusion and drops the detail that would have shown the mismatch. Reading the consuming code is cheap, and it is the only place that says what the setup should have been.

## Watch-outs

- Silent coercion is the usual cause: `Number`, `parseInt`, `Date` parsing, booleans read from strings. Every one of them turns a wrong format into a valid-looking default
- A second helper that only re-runs the first one's test will confirm the negative. Send it to read the code, not to repeat the run
- The same applies to your own negatives. A helper is only the case where you cannot see the setup

## Adjacent Patterns

- [[a-partial-read-proves-presence-not-absence]] is the same asymmetry in reading files: a narrow look can show something is there, never that it is not
- [[verification-needs-a-negative-control]] is the mirror: a check that cannot fail proves nothing, and a check that cannot pass proves nothing either
- [[an-unnamed-blind-spot-reads-as-an-empty-source]] is the general case, where a gap in what was looked at reads as an absence in what exists

## Source

A web app's "new since last visit" pill, tested by a browser-driving helper on 2026-10-07. The negative came from an ISO string where the code read epoch milliseconds, and a second helper found it by reading the code.
