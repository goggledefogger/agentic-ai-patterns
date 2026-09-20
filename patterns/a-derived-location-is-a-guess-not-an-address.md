---
type: pattern
date: "2026-08-19"
source: The dashboard PR #6 — a path-flattening formula copied to three call sites, wrong on every Windows machine since the first one, found by the first outside tester
tags:
  - code
  - cross-platform
  - drift
  - anti-pattern
---

# A Derived Location Is a Guess, Not an Address

Code that computes where something lives — flattening a path into a folder
name, building a cache key, predicting the file another tool writes — is
reimplementing a decision that belongs to someone else. The formula is a guess
about a foreign naming scheme, and it is wrong the moment that scheme meets an
input the author never typed.

Two things go wrong together, and the second hides the first.

## The Problem

A dashboard predicted where a CLI files its session transcripts by flattening
the project path into a folder name. The pattern handled `/` and `.`, which is
every character a macOS source path contains. A Windows path adds a drive
colon and back slashes, and the vault's own name added a space.

So on Windows every lookup missed. Not intermittently: on every machine, from
the first run, forever. And because the formula had been **copied to three
call sites**, one wrong guess presented as three unrelated bugs — empty chat
history, a gauge that never worked, and the member's own vault surfacing in a
list of other people's sessions. Three symptoms, three plausible separate
investigations, one root cause.

The cost lands entirely off the development platform. A derivation formula is
tested by the author's own filesystem, which is the one input guaranteed to
satisfy it.

## The Pattern

1. **One definition.** A formula that exists in three places drifts in three
   places, and its failure arrives disguised as three bugs. Name it once, call
   it everywhere.
2. **Derive, then *find*.** Treat the computed location as a first guess. When
   it misses and the target has an unambiguous name, search for it:

   ```js
   function transcriptFile(sessionId) {
     const name = sessionId + '.jsonl';                 // uuid, unambiguous
     const guess = path.join(FLEET_DIR, brainSlug(), name);
     if (fs.existsSync(guess)) return guess;
     for (const d of readDirs(FLEET_DIR)) {             // fall back to a search
       const hit = path.join(FLEET_DIR, d.name, name);
       if (fs.existsSync(hit)) return hit;
     }
     return null;
   }
   ```

   The fast path stays fast, and the fallback survives whatever the upstream
   does to its naming scheme next.
3. **The fallback needs an unambiguous target to be safe.** A uuid filename
   can be searched for; `config.json` cannot, because the first hit may belong
   to somebody else. No unique name means no fallback, and the derivation has
   to be correct instead.

## Why It Works

The formula and the fallback fail on different things. The formula fails on
inputs the author never saw, which is a permanent and growing set. The search
fails only if the file genuinely isn't there, which is the answer you wanted.
Keeping both means the cheap path handles the common case and the honest path
handles the case you couldn't predict, and neither has to be right about the
upstream's naming rules.

One definition is what makes that affordable. Three copies means three places
to add the fallback, so it gets added to the one that was reported.

## When to Use

- Any code predicting a path, key, or identifier that a *different* program
  owns and can change: transcript and cache directories, log locations, build
  outputs, slugified names.
- The moment you copy such a formula a second time. That is the signal, not
  the third copy.

## When NOT to Use

- The location is a value the upstream hands you (an API returns the path, a
  manifest names it). Use the value; don't re-derive one you were given, per
  `a-heuristic-where-an-exact-key-exists`.
- The search space is unbounded or the name isn't unique, per rule 3. A
  fallback that can return the wrong file is worse than a miss.

## Adjacent Patterns

- `a-heuristic-where-an-exact-key-exists` — the same lesson from the other
  side: the exact key was usually available and never looked for.
- `a-dry-run-that-derives-will-drift` — echo over derive, one level down; a
  second expression of one intent diverges from the first.
- `never-trade-the-screen-for-a-promise` — the surface failure this caused,
  and why it stayed invisible for so long.
- `portable-engine-local-profile` — where machine-specific facts belong when
  a formula shouldn't be guessing them at all.

## Source

The dashboard PR #6, merged 2026-08-19. The slug formula became a
single `brainSlug()` flattening separators, the drive colon and whitespace
together; `transcriptFile()` guesses that path and then walks the sessions
root by uuid filename when the guess misses. First outside contribution to the
repo, from the first outside tester's own agent on her first day, carrying
back a correctness rule the authors' platform could not have taught them.
