---
type: pattern
date: "2026-10-02"
source: The dashboard's skill store, installed by the one person who builds the app, publishes the skill and uses both on the same machine
tags:
  - dogfooding
  - triage
  - product
  - anti-pattern
---

# The Author Sees Bugs No Member Will

When the person testing a product also wrote it, published what it installs and runs it on the machine that holds the source, a good share of what looks wrong is only wrong for them. Fix it all and you ship changes for a user who does not exist, and sometimes a rebuild that solves nothing for anyone else.

## The Problem

A dashboard has a store that installs agent skills for members who are not developers. Pressing Install sends a fixed message into a chat and an agent does the install. The owner installed one himself and found the end confusing.

He was the app's developer, the skill's publisher and the user, all on one machine. About half of the strange parts came from that:

- The agent said the skill marketplace was "already added". It was, because he publishes it. A member adds it fresh
- The agent went reading the app's own source code to understand the install. The checkout was sitting right there. A member has no checkout

The install itself had worked, in 38 seconds. The one thing every member would also see was the agent's closing reply, a recap of a plan they had read seconds earlier (see `enforce-the-message-not-the-field.md`). He had said up front "don't overcorrect because maybe it worked just fine".

## The Pattern

Before fixing anything, sort each symptom into one of two columns:

| Only the author sees this | Every member sees this |
|---|---|
| State left by building or publishing (already registered, already cached, already signed in) | What the product says and does on a clean machine |
| The agent wandering into source that sits beside it | The words, steps and timing of the normal path |

Fix the right column. For the left column, write down why it is author-only, so the next session does not rediscover it as a bug.

Here the sort left exactly 1 change worth making, one sentence in the fixed message. It also ruled out a rebuild of the install as a button with no chat, which would have chased symptoms only the author had.

## Why It Works

A dogfooding author is the cheapest tester and the least typical one. Their machine carries every leftover of building the thing, and their eye is tuned to internals a member never sees. The sort turns "it felt off" into a short list of real defects plus a note on the rest, instead of a rewrite aimed at the wrong person.

## When to Use

Any time the reporter is also the builder or the publisher, and especially on a single machine. The tell is a symptom that mentions something only a builder has: a source checkout, a publish step, a dev account, a cache from yesterday's build.

## When NOT to Use

When the author-only symptom points at a real gap a member could reach another way. An agent reading source because it was nearby may mean the instructions were thin. Check that before filing it as author-only.

## Adjacent Patterns

- `the-second-principal-is-invisible-to-a-reader-built-for-the-first.md` is the reverse case: a system built for one person hides the second. Here the one person is the builder, and their view hides what the member sees
- `open-intent-funnels-closed-acts-in-place.md` came out of the same kind of live dogfooding, and is the rule that seemed to argue for the rebuild, since an install looks like a closed act

## Source

A member's second-brain dashboard, 2026-10-02. The store's install was tried by its own author, and sorting the symptoms by audience left one sentence to change.
