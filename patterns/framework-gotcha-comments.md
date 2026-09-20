---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo)
tags:
  - code-comments
  - debugging
  - framework-knowledge
---

# Framework-Gotcha Comments

When fixing a bug that came from a framework-specific behavior you didn't know about, leave a 2-line comment above the fixed line explaining *why* the naive approach fails.

## The Problem

Framework-specific gotchas (Vite's static env replacement, Next.js hydration rules, Tailwind purging, Webpack dynamic imports, Flask's route-matching order) cause the same bug over and over in the same codebase. The author fixes it, forgets why, and writes the naive version again three months later.

The usual "default to no comments" rule applies to WHAT the code does — well-named identifiers cover that. But *why a non-obvious pattern is used* is exactly the kind of comment that pays for itself.

## The Pattern

When fixing a framework gotcha:

1. Write the fix.
2. Add a 2-line comment above the fixed line explaining the underlying constraint. Not what the fix does — *why the naive approach fails*.
3. Commit them together. If you wait, you'll forget the gotcha by the next commit.

## Example

From the participant's cockpit at `cockpit/src/components/AuthGate.tsx`:

```ts
// Vite statically replaces import.meta.env.VITE_* at build time.
// Dynamic key access (import.meta.env[key]) does NOT work.
const EXPECTED_HASH: string = import.meta.env.VITE_APP_PASSWORD ?? "";
```

A new reader who doesn't know Vite's static-replacement behavior would look at that literal key access and ask "why didn't they just use a constant?" The comment pre-empts the question.

## Writing the Comment

Good examples:

```ts
// React 18 double-invokes effects in Strict Mode. The cleanup runs before the
// second mount, so subscriptions must be symmetric.
```

```python
# SQLite WAL mode doesn't allow cross-connection reads until a checkpoint fires.
# Keep writes on one connection and reads on another, or force checkpoint.
```

```js
// Tailwind only includes classes it can see as complete string literals at build
// time. Interpolated names (`text-${color}`) get purged. Use a safelist instead.
```

Bad examples (describe what, not why):

```ts
// Set the hash from env
const EXPECTED_HASH: string = import.meta.env.VITE_APP_PASSWORD ?? "";
```

```ts
// Use static key access
const EXPECTED_HASH: string = import.meta.env.VITE_APP_PASSWORD ?? "";
```

## When to Use

- You just fixed a bug and the fix looks counterintuitive.
- A future reader would reasonably write the naive (wrong) version if they didn't know the gotcha.
- The gotcha is framework-level, not business-logic-level.

## When NOT to Use

- The code is self-explanatory.
- The "gotcha" is actually a code smell that should be refactored, not commented around.
- The comment would need to be longer than ~3 lines. At that point, write a decision doc or link to the framework issue tracker.

## Adjacent Patterns

- Pairs with decision-doc pattern when the gotcha drives a structural choice (e.g., "we chose X framework despite this gotcha because Y").
- Add the gotcha comment discipline to your global `~/.claude/CLAUDE.md` so it applies across every project.
