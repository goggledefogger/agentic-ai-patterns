---
type: pattern
date: "2026-08-17"
source: The personal agent's dispatch ledger — metered every headless worker call from 2026-08-07, and nothing read it until 2026-08-16, when the question it was built to answer ("is delegating actually cheaper?") had to be answered from scratch
tags:
  - monitoring
  - observability
  - anti-pattern
  - instrumentation
---

# Instrumentation With No Reader Is a Decision to Collect Evidence and Never Look

Adding metering is the satisfying half. It is cheap, it is obviously correct, it goes in during the change that motivated it, and it makes the system feel measured. Then nothing ever reads it, and the system is exactly as unmeasured as before, with a file that grows.

## The Problem

A dispatcher recorded one JSON line per delegated call — tokens, cost, turns, duration, denial counts, deliberately no worker text. The design was careful: it reasoned explicitly about which fields could leak content and dropped those. It was written, reviewed, and committed as part of getting delegation right.

Ten days later, asked whether delegating work to a subagent was actually saving anything, the answer was *"nobody can tell"* — and the reason was not missing data. **Two rows sat on disk and the writer was the only file in the repo that mentioned the path.** Nothing aggregated them, nothing surfaced them, no command printed them, and no scheduled job touched them. The one place that referenced the ledger was the code that appended to it.

The failure is quiet in a specific way: a write-only ledger passes every review. The writing is correct. The schema is sensible. Nothing errors. It looks like observability right up until someone asks the question it exists to answer.

Related shapes, same root:

- A metrics endpoint nothing scrapes
- A structured log field no query uses
- An audit trail with no report, which is worse than none because it implies review that is not happening
- A health check that exists as a subcommand nobody runs on a schedule — detection without a schedule is not monitoring, it is a command nobody types
- A cron whose `>> file 2>&1` redirect is the only place its output and exit code go. Fresh scar (the household agent's Pi, 2026-08-17, same day this pattern was written): a poller lost its credential and printed `FATAL` + exit 2 every 10 minutes for 11 hours — into a log that cron redirected everything into, that the error-alerter didn't tail, and that no incident pattern named. The guard fired perfectly the whole time; the redirect was a reader-shaped hole. Detection that terminates in an unread file is the subcommand-nobody-types with a cron schedule attached

- A content hash recorded per record that nothing ever compares. Fresh scar (the household agent's wiki, 2026-08-21): `.state.json` stored a `sha256` for every immutable Layer-1 capture from the day the store was designed. `grep -rn file_sha256` over the whole `scripts/` tree returned **two writes and the function definition** — no reader, in fourteen months. A sync job had meanwhile been overwriting one capture in place on every upstream revision, and the state file held the proof throughout: two rows on the same path with different hashes

## An unread field also stops meaning what you think it means

The scar above has a second half that is easy to miss, and it is the more useful one.

When the reader was finally written — compare each recorded `sha256` against the file on disk — it fired on **seven** captures. Five were real. Two were not: an ingestion that writes a raw file, records its hash, then reformats that same file seconds later leaves a hash of an intermediate version. Nothing was destroyed; the field simply never described the final bytes. A 2-in-7 false-positive rate, on the first run, on real data.

That drift is not sloppiness — it is the predictable consequence of the field being unread. **A field nobody compares has nothing holding its meaning still.** No test asserts what it should equal, no caller breaks when a writer records it a moment too early, and every refactor is free to move the write. By the time a reader arrives, the semantics have quietly moved and the reader inherits the drift as false positives.

The fix was not a better hash. It was noticing the wiki was already a git repository, and that `git log --diff-filter=M -- raw/` answers the actual question — *was a capture changed after it was committed* — exactly, with no second store to keep honest. The purpose-built field had drifted; the general-purpose one could not, because everything else in the system depends on it being right.

Two things follow:

- **Before building a reader for an old field, check the field still means what its name says.** Validate the reader against known-good and known-bad real records, not fixtures. A reader shipped on a drifted field produces confident wrong answers, which is worse than the silence it replaced.
- **Prefer an instrument something else already depends on.** Version control, the filesystem, the queue's own offsets — these stay correct because other things break when they don't. A field that exists only to be read someday is load-bearing for nothing, which is exactly why it drifts.

## The Pattern

**Ship the reader in the same change as the writer, or do not ship the writer.**

1. **Name the question before adding the field.** "Is delegating cheaper than doing it inline?" is a question. "Let's record token counts" is a habit. If you cannot state the question, the field is speculative and speculative fields never get read.
2. **A reader is a command, and it takes an argument you would actually type.** `--days 7`, `--resource X`. If the only way to read it is `jq` improvised at the prompt, it has no reader.
3. **Wire it to something that already reaches a human.** A report nobody runs is the same failure one level up. Fold the signal into an existing daily digest or alert rather than creating a second surface — a new channel is another thing that needs a reader.
4. **Make the empty case say so.** A reader over zero rows must print `UNSCORED` / `no data` and state that this is **not** a pass. Rendering an empty ledger as green is how a broken writer stays hidden.
5. **Report gaps as gaps.** If a row lacks a cost, print `unavailable` — never `0`, never an inferred value. A reader that invents numbers is worse than no reader, because its output gets believed.

## Why It Works

The writer and the reader fail at opposite times. A writer fails loudly at write time and is easy to test. A reader fails at *question* time, which is months later, under pressure, when the person asking has no budget to build one. Shipping them together moves the reader's cost to the moment when someone actually understands what the data means.

It also forces the honest scoping conversation early. Writing "we'll figure out how to read it later" is usually a sign the question was never sharp, and a sharp question often reveals that two of the four recorded fields are enough — or that the whole thing should be a counter.

## Trade-offs and Limits

- **Not every write needs a reader in the same commit.** A raw event log feeding an existing pipeline already *has* a reader. The rule bites when you are inventing a new store.
- **A reader is a maintenance surface too.** Keep it arithmetic over what is there, with no scoring logic that can drift from what the writer means.
- **Beware the reader that is only a dashboard.** A page someone must remember to open is a weaker reader than a line in a digest that arrives whether or not they thought to look.

## Adjacent Patterns

- [[opaque-write-needs-a-read-back]] — there the receipt cannot disconfirm the payload; here the payload is recorded correctly, drifts unobserved, and is never consulted.
- [[unrun-checks-read-as-passing]] — a check that never ran; this is a check never written, for data already on disk.
- [[green-tests-can-mirror-the-same-guess]] — why a reader must be validated against real records rather than fixtures built from the same assumption.
- [[llm-wiki-maintenance]] — the architecture whose immutable-raw invariant this scar broke.
