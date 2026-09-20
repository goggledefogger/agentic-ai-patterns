---
type: pattern
date: "2026-08-04"
source: A personal agent's fleet-status script — two heuristics shipped and caught in one evening; corroborated by the CLI's own 64KB tail read of session transcripts (anthropics/claude-code #26240, #26249, #65361)
tags:
  - code
  - debugging
  - anti-pattern
  - verification
---

# A Heuristic Where an Exact Key Exists

One 400-line script, one evening, two bugs with the same shape. Both were reasonable-looking approximations standing in for an exact answer that was available the whole time and never looked for.

**Which row is me?** A fleet sweep marked its own session with `<- me` by tagging *the first active transcript in a project this process has a pid in*. Correct while one session per project is live. With two, the freshest wins — and the freshest was the other session. The exact key: the harness exports `CLAUDE_CODE_SESSION_ID`, and its value **is** the transcript filename stem. Identity was a string compare.

**What is this session called?** The name is written into the transcript as its own record, wherever `/rename` happened — observed at line 180 of a thousand-line file. Reading the tail (the window that is correct for "the last thing the user said") returns nothing, which renders as *this session has no name*. The exact answer: scan the file, keep the last such record.

Neither failed loudly. One marked the wrong row; the other showed a blank indistinguishable from an honest "never named."

## Why the proxy is attractive

You reach for a heuristic when you believe the exact key does not exist — and that belief is usually untested. Nobody checked `env` for a session id. Nobody asked *when is this record written* before reusing a reader that already parsed the file. The proxy does not feel like a compromise at the time; it feels like the only option, because the search for the real one never happened.

The tell is a comment justifying the approximation. Ours read *"the freshest active transcript in its own project is almost certainly the sweep itself"* — an accurate description of a guess, written confidently enough that nobody revisited it. **A comment defending a heuristic is a note that the exact key was never sought.**

## The severity is inverted

Both bugs fail on exactly the cases the tool exists for.

The identity heuristic is correct until a second session appears — and a *fleet* sweep's entire purpose is multiple sessions. The tail read is correct until a transcript outgrows the window — and long sessions are the ones people bother to name and later need to find. A heuristic degrades with load, size and contention, so it works in testing and on quiet days, then misleads precisely when the stakes rise.

This is not hypothetical elsewhere either: Claude Code's own session list reads only the last 64KB of each transcript for the title record and loses names on long sessions for the same reason ([#26240](https://github.com/anthropics/claude-code/issues/26240), [#26249](https://github.com/anthropics/claude-code/issues/26249), [#65361](https://github.com/anthropics/claude-code/issues/65361)). A shipped implementation and a naive reimplementation made the same call independently — the mark of an attractive mistake, not a careless one.

## Sighting three: a person's name as a search key (2026-08-31)

Asked to file "the email reply i sent to a family member", an agent searched Gmail for `in:sent <name>`. Gmail has no person operator, so a bare name is a full-text body match: it returns the mail that *mentions* her and drops the rest. Her address is `family-member@example.com`. The agent then offered a four-option "which one did you mean?" list with a real send missing from it, and nothing marked the list partial.

The exact key was one call away and had been the whole time — `who_is.py "<name>"` returns her People-file handles block, `email=family-member@example.com`, and the resolver already existed. The skill that wrote the query simply never pointed at it.

**Same shape, third surface, four weeks:** session identity (08-04), a GitHub handle (08-18), an email address (08-31). The 08-18 one is the sharpest, because it is recorded *in the very skill that owns handles* — the personal agent read her People file, saw no field named `github`, and asked the author, "with `family-member@example.com` sitting right there." **A pattern written down in one skill does not reach the skill that needs it.** Put the rule where the guess gets made, not only where the knowledge lives.

The inverted severity holds exactly: a name-as-keyword query succeeds when someone's address contains their name and fails for everyone else — right on the easy case, wrong on the general one.

## The Pattern

**1. Spend sixty seconds looking for the exact key before writing the proxy.** `env | grep -i <tool>`, the ids already in the data, a manifest, the record's own fields. The cost of looking is trivial against a wrong answer that never raises.

**2. Ask when a record is written before choosing where to read.** The window is an assumption about the writer, disguised as an optimization:

| When it is written | Correct read |
|---|---|
| Always last of its kind | tail window — cheap and correct |
| Once, at an arbitrary point | **full scan** — a window is a coin flip |
| Repeatedly, latest wins | **full scan**, keep the last hit |
| At a known offset | head read |

**3. One file may need two strategies, and that is fine.** The tail read stayed for the last user message because it is genuinely correct there. Do not unify them for tidiness; annotate each with *why* that window.

**4. Prefer a measurable cost to an invisible one.** A streaming full scan with a cheap pre-filter is I/O bound and fast enough. Pay milliseconds you can measure rather than a correctness bug you cannot see. (Keep the pre-filter a filter — the value must still come from a real parse.)

## Verification: the fixture must contain the contended case

Both bugs survive a naive test. One session in the fixture and the identity heuristic is right; a short fixture and every record sits in the tail. **The fixture has to be built to fail the approximation:**

```python
# two sessions where the FRESHEST is deliberately not me
os.utime(other, (now - 5, now - 5))     # freshest
os.utime(mine,  (now - 90, now - 90))   # older = me
assert rows[mine_id]["is_me"] and not rows[other_id]["is_me"]

# and the record pushed well outside any tail window
named.write_text("\n".join([filler, title_a, title_b] + [filler] * 60))
assert session_title(named) == "renamed-later"
```

Then the negative controls, or a function that always returns something also passes: with no session id, **no** row may claim to be me; with no title record, the name must be empty rather than a placeholder.

## Adjacent Patterns

- [[the-guessable-path-is-the-stale-one]] — the sibling: there you read the wrong file, here you read the right file through the wrong window, or identify the right thing by the wrong key.
- [[a-fast-answer-is-a-suspect-answer]] — truncation that produces fast wrong *output*. Here it produces a confident *absence*, which is quieter.
- [[unrun-checks-read-as-passing]] — "did not run" and "ran and passed" share a shape, exactly as "no name" and "name outside the window" do.
- [[decorative-gate]] — same family of question: show me the input that makes this wrong, and the test that proves it.
