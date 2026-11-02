---
type: pattern
date: "2026-11-01"
source: A dashboard that tracks background work an agent announces by writing a small marker file. A chat whose work had started minutes earlier was shown as running for 365 days. The agent had typed the start time by hand and landed exactly one year off, copying the year from the numbers in the instruction doc's examples
tags:
  - agents
  - prompts
  - hallucination
  - verification
  - timestamps
---

# Literal Values in Agent-Facing Examples Get Copied

An instruction doc that shows a JSON example with a real-looking number teaches the agent that number's shape, including its year. When the agent has to fill the same field, it writes something near the example instead of reading the clock. The result is a value that is well formed, plausible, and wrong.

## The failure

A dashboard lets an agent announce background work by writing a marker file with a `since_ms` field, epoch milliseconds for when the work started. The doc described the field in one phrase and showed two examples with literal numbers from the previous year.

The agent filled the field by hand. It landed on the same date one year earlier, so the dashboard said the work had been "running 365d ago, and may be left over" under a chat whose work had started minutes before. Nothing errored. The marker parsed, the field was a number, and the number looked like every other timestamp.

## The pattern

Two layers, because prose lowers the error rate and cannot take it to zero.

1. **Fix the instructions.** Say where the value comes from: read it from the clock (`date +%s000` or `Date.now()`), never type it. Call every literal in an example a placeholder, in those words. Prefer examples that show a command or a token over a number the agent could mimic
2. **Check the value against a signal the system already has.** The server compares the agent-written time with facts it can see on its own: a start time in the future is impossible, and one more than a day older than the marker file's own save time cannot be right. Either case falls back to the file's save time. This catches the miss for every agent and every engine, including ones that never read the doc

The general rule: an agent-supplied fact that the system can cross-check against its own signal (file time, clock, git history, process table) should be checked there and not trusted. Anything an agent types from memory is a guess about the world, and the world is usually one lookup away.

## Why It Works

The prose fix removes the cause for agents that follow it. The cross-check removes the dependence on the agent following anything. Each covers the other's gap, and the second is cheap because the signal already exists.

## When to Use

- Any field an agent fills that has a ground truth the host can read: times, ids, paths, counts, versions
- Any instruction doc with example values, which should be audited for numbers, dates, and names an agent could echo

## When NOT to Use

When no independent signal exists, a fallback only hides the error. Then the better fix is to take the field away from the agent and have a script write it.

## Adjacent Patterns

- **`a-required-slug-invents-the-missing-half.md`**: the same family, a slot gets filled with something plausible because the format demands a value
- **`verdicts-from-structured-signals.md`**: judge from the signal the system owns, not from what the agent says

## Source

A dashboard background-marker incident on 2026-11-01, fixed in a dashboard PR the same day.
