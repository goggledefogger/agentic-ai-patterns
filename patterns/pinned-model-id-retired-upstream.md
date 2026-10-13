---
type: pattern
date: "2026-10-09"
source: A dashboard that routes chats to several model vendors; one vendor retired a model and began serving its successor under the old name the same day, with no shutdown date
tags:
  - agents
  - models
  - lifecycle
  - drift
  - reliability
---

# A Pinned Model ID Retired Upstream

A dashboard that runs chats on several model engines had a good procedure for a new model. A seed list of rows, a `supersedes` field so a pick stored on a replaced id read forward to its replacement, a panel line naming a newly shipped model nobody had added a row for yet, and a test that goes red when an offered id is not documented. None of it covered the other direction.

Then a vendor's notice arrived: a model was deprecated, no action required, all requests to the old name are automatically redirected to the successor at the same price. The vendor's model list carried no field saying so. The engine with a live list simply stopped offering the id. A pick stored on it had no handled path. The runtime "model gone" detector matched one vendor's wording and fell back to a hardcoded cheap model. And because of the redirect, every record that said the old model made something was now wrong, from that day, with nothing erroring to say so.

## Why it is missed

- A release announces itself. A retirement is an email and a page.
- Only 1 vendor's model list carries a machine-readable date (OpenAI's `shutdown_date`). A public catalog carries a coarse `status: deprecated` with no date. The others carry nothing. The row has to be typed in by a person from the notice.
- A redirect hides the failure completely. Nothing errors, so no detector fires, and the name lies in every save.
- Supersede-forward covers the upgrade case only. The successor's row lists what it replaces, so a model with no successor row (a different vendor, an engine whose list is live) has nowhere to read forward to.

## The pattern

**3 states, not 2.** Active, deprecated, retired.

- **Deprecated**: still offered, marked with its retirement date and successor. A pick runs what was picked, after 1 nudge. Never swap a model under someone mid-work.
- **Retired** (the date has passed, or the name is redirected, or another row supersedes it): never offered. A stored pick reads forward on use, at request time, keeping its lane prefix and effort suffix.
- **A redirect is retired from day 1**, because the name lies.

**4 fields on the row**, flat, beside the upgrade fields: `deprecated` (date announced), `retires` (date, absent when unannounced), `successor`, `source` (the vendor's public page, never a mail link), plus `redirected` when it applies. For an engine whose list is live, the row is lifecycle-only: the live list says what exists, the row says what is leaving.

**Tell the person where it is pinned.** A "leaving" line beside the "new model shipped" line, 1 per pin (the default, a member overlay, a ritual, a helper constant, a store listing), with a Move control where the pin is a file the tool owns and a plain sentence where it is code.

**A runtime refusal is the backstop, not the signal.** Match every vendor's wording, require the model id in the message (a 404 for a bad URL is not a retired model), fall to the row's successor, then the lane default, then the old hardcoded fallback, and record a dated observation that expires or can be dismissed. Never retire a row from an error.

**Tests that go red**: a retired id named anywhere by exact string, longest match, with effort suffixes and sentence punctuation included; a malformed date or a mail-link source; a successor that is itself retired. Plant an id and watch the string test fail before trusting it.

## Anti-patterns the first build had

Each of these was in the first version and caught in review.

- **Following the successor regardless of state.** "Use it anyway" on a deprecated model quietly ran the successor.
- **Normalizing ids too eagerly.** Stripping the lane prefix for the new path made the old upgrade path rewrite a routed id into a slug that may not exist. Keep the upgrade path on bare ids; let only a lifecycle `successor` move a prefixed one.
- **A UTC day boundary.** A model retired at 4 pm the day before for a member on the west coast.
- **Overwriting the member's own overlay** to record an observation when the file would not parse.
- **Recording a refusal against the model you are on now** when an old chat reloaded and re-rendered its error.
- **Matching "not found" alone.** A missing endpoint, a bad URL, and a missing local file all counted as a retired model, and a hit now changes the model and writes a record.

## Neighbours

- [`self-reporting-staleness-check.md`](self-reporting-staleness-check.md): the stale-string test is the rule you add at the moment of the retirement, and delete when the id is gone from the corpus.
- [`stale-pointer-asserts-confidently.md`](stale-pointer-asserts-confidently.md): the pinned id is the pointer; a redirect is the pointer asserting confidently.
- [`a-deferred-fallback-decides-at-fire-time.md`](a-deferred-fallback-decides-at-fire-time.md): read forward at request time, never at load.
- [`live-reference-outlives-its-owner.md`](live-reference-outlives-its-owner.md): a pinned id outlives the model it names.
- [`local-model-agentic-tool-calling.md`](local-model-agentic-tool-calling.md): a runner's model list is a stale catalog.

## How to adopt

1. The day a notice arrives, add the row with the 4 fields from the vendor's public page, not from memory and not from the email's tracking links.
2. Make the picker show deprecated and hide retired, and read forward at request time.
3. Inventory the pins and show them beside the new-model line, with Move where you own the file.
4. Broaden the refusal matcher to every vendor you route to, require the model id in the message, and make its fallback the row's successor.
5. Add the tests, then plant a retired id to prove the string test fails.
6. Read the machine signals that exist (`shutdown_date`, a catalog's `status`) as prompts to add a row, never as permission to retire one.
