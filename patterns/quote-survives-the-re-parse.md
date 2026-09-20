---
type: pattern
date: "2026-08-05"
source: The household-agent repo deploy-to-pi.sh — `ssh host "cmd '$description'"` and a prose field containing "the Pi's". Second instance 2026-08-26: `onclick="fn('$title')"` in the Hub, where the value crosses an HTML attribute parser AND a JS string parser
tags:
  - code
  - shell
  - anti-pattern
  - operations
---

# A Value That Crosses a Re-Parse Boundary Must Be Quoted At It

`ssh host "cmd '$value'"` looks quoted. It is quoted **locally** — and that is
the half that does not matter. The remote shell receives one flat string and
parses it from scratch, with no memory of which characters you thought were
delimiters. Any quote character inside `$value` is a delimiter on the far side.

## The Problem

A deploy shipped cron jobs to a Raspberry Pi:

```bash
ssh host "bash automation-registry.sh add-cron '$schedule' '$command' '$description'"
```

Thirty-one jobs, thirty of them fine. The thirty-first had a description that
had grown, over months of incident notes, into prose:

> ... Kept the Pi's durable log dir, kept the repo's `--hours 8` (the script's
> default is 72h) ...

Three apostrophes. The first one **closed** the single quote, and the rest of
the sentence — parentheses included — arrived as shell syntax:

```
/bin/bash: -c: line 1: syntax error near unexpected token `)'
```

Nobody had touched the quoting; someone had edited a **comment**. The failure
was invisible for the same reason it was possible: a description field feels
like documentation, so it is written the way English is written, and English is
full of apostrophes.

The escalating detail: the repo's own house rules already carried this exact
warning — *"Never use heredocs for commit messages or config edits. Apostrophes
and dollar signs break them."* The rule was true, written down, and had never
travelled to the new call site, because the new call site was `ssh`, not a
heredoc. **A hazard rule attached to a mechanism does not protect the next
mechanism with the same hazard.**

## The Pattern

**Escape at the boundary, mechanically, for every value — not just the one that
looks dangerous.**

```bash
# POSIX single-quote escape (' -> '\''), bash 3.2-safe.
shell_quote() { local s=${1:-}; printf "'%s'" "${s//\'/\'\\\'\'}"; }

ssh host "bash registry.sh add-cron $(shell_quote "$schedule") \
                                    $(shell_quote "$command") \
                                    $(shell_quote "$description")"
```

Three parts:

1. **Quote every field, not the prose one.** The schedule carries globs
   (`0 */6 * * *`) and the command carries `&&` and redirects. Today they are
   safe by luck — no apostrophes — and that is not a property anyone is
   maintaining.
2. **Test through the boundary, not against a string.** A test that compares
   `shell_quote "$x"` to an expected literal passes while proving nothing about
   the far side. Run the result through a fresh shell, which is what SSH does:

   ```bash
   _roundtrip() { bash -c "printf %s $(shell_quote "$1")"; }
   assert_eq "$prose" "$(_roundtrip "$prose")"
   ```
3. **Pin the bug, not just the fix.** Assert that the *old* naked-quote form
   still fails on the real value (`rc == 2`). Without it, a future refactor
   that quietly reverts the quoting leaves a green suite.

## Second instance — 2026-08-26, two parsers inside one HTML attribute

The same shape, in a browser, with one extra twist that the shell case does not
have. A Hub row built its buttons like this:

```js
`<button onclick="castStream('${escapeJSString(row.name)}', 'movie')">Cast</button>`
```

and `escapeJSString` escaped exactly what a JS string literal needs, backslash
and apostrophe. The value still crosses **two** parsers, and they peel in a
fixed order: the browser HTML-decodes the attribute first, then hands what
survives to the JS parser. So a double quote in `row.name` closes the `onclick`
attribute before a single character of JS has run, and no JS-level escape can
prevent it — `\"` is not an escape to an HTML parser, only `&quot;` is. Real
titles do this (`"Crocodile" Dundee`), and these names were derived from torrent
release names, which are not ours to trust.

Two additions to the pattern fall out of it:

4. **Escape the layers in reverse order of how they are peeled, and know which
   layer each character belongs to.** JS-escape first, then HTML-encode, and
   encode `&` before the entities you are about to introduce or `&quot;` becomes
   `&amp;quot;`. An escape applied at the wrong layer *looks* like escaping and
   does nothing, which is worse than none at all.
5. **If the boundary is optional, delete it instead of quoting across it.**
   `ssh` gives you no choice, the second parse is the mechanism. A browser does:
   put the value in a `data-*` attribute and have the handler read
   `this.dataset`, and there is exactly one parser, which the existing
   attribute-escaping helper already handles correctly. No nesting, no ordering
   rule, nothing to get subtly wrong later. The fix that survives review is the
   one that removes the class rather than the instance.

### It also confirms rule 3, by way of violating it

The new test round-tripped the value through an HTML decode and a JS literal and
asserted it came back intact. Every case passed. Run against the **old, broken**
implementation, every case still passed — because a bare `"` inside a
single-quoted JS literal is perfectly legal, so the round trip exercised only
the JS layer and was blind to the vulnerability by construction. The assertion
that actually failed on the old code was the blunt one: *the output must not
contain a bare double quote at all*.

Rule 3 above already said this — pin the bug, not just the fix — and writing the
round trip felt so much like thoroughness that the rule did not come to mind
until the old implementation was run against the new test on a hunch. Round-trip
tests are necessary and not sufficient: they catch layer-ordering breakage,
which the blunt assertion misses, and miss termination entirely, which is the
thing that gets you owned. Keep both, and say in the test file which one is
which, or the next reader deletes the ugly one as redundant.

## When It Shows Up

Every boundary where a string is re-parsed by a second interpreter: `ssh
host "…"`, `bash -c`, `sh -c` in a Dockerfile or systemd `ExecStart=`, `eval`,
`su -c`, `ansible` raw modules, `subprocess` with `shell=True`, a cron line
assembled by a script, SQL built by concatenation, JSON embedded in a
string field, a shell command inside a YAML `run:` block, and any inline event
handler in generated HTML (`onclick="fn('…')"`, `href="javascript:…"`), where
the two parsers sit inside a single attribute.

The tell is **nested quote characters in a single line**, especially with a
`$variable` between them. If you can see `"` and `'` in the same command with
an expansion inside, there is a re-parse and it is unguarded.

The second tell is a **free-prose field being interpolated at all** —
descriptions, commit messages, incident notes, PR bodies, alert text. Those are
where apostrophes live, and they are exactly the fields reviewers skim past
because they read as documentation rather than as data.

## Related

- [[tolerance-named-the-failure-not-the-benign-set]] — the guard that hid this
  one for months
- [[unrun-checks-read-as-passing]] — the sibling shape: a step that silently
  did not happen
- [[framework-gotcha-comments]] — recording a hazard where the next author
  will hit it
- [[one-authority-for-repeating-behaviors]] — one escaping helper, not one per
  call site

## The Rule of Thumb

If a string will be parsed by an interpreter you are not currently standing in,
quote it with a function, test it through the real thing rather than against an
expected literal, and assume the prose field contains an apostrophe — because
eventually someone will write one. Then ask whether the second parse is load
bearing at all: if it is not, remove it, and the whole class goes with it.
