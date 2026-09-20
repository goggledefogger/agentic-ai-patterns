---
type: pattern
date: "2026-07-31"
source: A work skills repo, 18 agent-authored commits that silently lost their Co-Authored-By trailer
tags:
  - git
  - agent-attribution
  - silent-failure
  - verification
---

# Commit Trailers Must Be Contiguous

Git recognizes a trailer block only when it is one unbroken run of lines at the very end of a commit message. A blank line inside that run splits it, and everything above the split silently stops being a trailer.

## The Problem

Claude Code is told to end commit messages with a co-author trailer, and often a session-link trailer alongside it. The natural way to write two trailers with the multiple-`-m` pattern is one flag each:

```bash
git commit \
  -m "subject" \
  -m "body paragraph" \
  -m "Co-Authored-By: Claude <noreply@anthropic.com>" \
  -m "Claude-Session: https://claude.ai/code/session_01ABC"
```

Every `-m` becomes its own paragraph with a blank line between. That blank line lands between the two trailers and breaks the block. Git then parses only the last run, so `Claude-Session:` is the trailer and `Co-Authored-By:` quietly demotes to body prose. GitHub renders no co-author.

What makes this expensive is that it fails invisibly from both directions. Nothing errors at commit time. And the line is still sitting there in `git log`, spelled correctly, in the position you expect, so a human reviewing the message sees a perfectly good trailer. You only find out by asking a parser, or by noticing months later that the avatars never showed up.

## The Pattern

Put every trailer in ONE final `-m`, separated by a literal newline inside the quoted string:

```bash
git commit \
  -m "subject" \
  -m "body paragraph" \
  -m "Co-Authored-By: Claude <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01ABC"
```

Then verify with git's own parser instead of by eye:

```bash
git log -1 --pretty=%B <sha> | git interpret-trailers --parse
```

That prints what git actually recognizes. Anything you wrote as a trailer and do not see listed is not a trailer.

## Why It Works

Reading the message tests whether the text looks right. Parsing it tests what every downstream consumer sees, which is the thing you actually care about. The check costs one command and turns a class of silent failure into a visible one.

## When to Use

- Any repo where an agent writes commits and attribution matters
- Any commit carrying more than one trailer, which is the only case where the bug can fire
- Once, retroactively, on a repo where attribution has never been verified, because the failure leaves no trace anywhere else

## The Cost of Finding Out Late

A trailer can only be fixed by rewriting the commit. Once those commits are pushed and other branches fork from them, the repair needs a force-push.

In the case that produced this pattern, 18 commits on a shared internal repo had lost their co-author. Fixing them meant rewriting published history that an active PR forked from, which would have dumped 2,000 lines of unrelated work into a colleague's open diff. They stayed broken. Only the forward rule changed.

That asymmetry is the argument for verifying early. The check is free before the push and unaffordable after.

## Adjacent Patterns

- Pairs with `verification-needs-a-negative-control.md`. A trailer that renders correctly proves nothing until you have seen the parser reject a broken one
- Put the rule in the global `~/.claude/CLAUDE.md` git section rather than a single project's, since the instruction that causes it is global

## Source

Found 2026-07-31 auditing agent attribution on a work skills repo. The bd-roadmap commits carried a single trailer and parsed fine. The discovery-brief commits carried two, written as separate `-m` flags, and `git interpret-trailers --parse` returned only the session link.
