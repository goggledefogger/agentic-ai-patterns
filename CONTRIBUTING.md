# Contributing

Patterns here come out of real incidents. A pattern that has not cost anybody
anything yet is a guess, and guesses are the one thing this repo has no room
for.

## The gate

Every file, yours or a fix to someone else's, has to clear four checks:

1. **No personal data.** No real names, no email addresses, no phone numbers,
   no account handles, no employer or client names. If the lesson only makes
   sense with a person in it, use a placeholder name.
2. **No private paths.** No absolute home directory with a real username in
   it, no home paths outside the ordinary tool locations, no private
   repository slugs or issue links.
3. **It works for someone who is not the author.** A reader with a different
   toolchain should still be able to act on it. Name the mechanism, not your
   setup.
4. **MIT.** By opening a pull request you agree your contribution ships under
   the license in `LICENSE`.

## Shape of a pattern

One lesson per file, in `patterns/`, named for the lesson in
`lowercase-with-hyphens.md`. Copy the frontmatter from any existing file:
`type: pattern`, a `date`, a neutral one-line `source`, and `tags`. Then the
body: what the pattern is, why it holds, when it shows up, and where it came
from.

Keep the source line neutral. "A deploy script reset a live session on every
run while its own log said otherwise" is provenance. A repo name and a commit
hash is not.

## Workflow

Fork `goggledefogger/agentic-ai-patterns`, branch, and open a pull request
against `main` with a clear description. One new pattern is a small PR and
lands quickly. A batch, a rewrite, or a change to the guides gets a closer
read. No force-pushing to `main`.
