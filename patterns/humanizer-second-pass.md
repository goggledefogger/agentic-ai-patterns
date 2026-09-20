---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo)
tags:
  - writing
  - voice
  - skills
---

# Humanizer Second-Pass

After editing text with a voice guide (positive rules for *your* style), run the `humanizer` skill as a second scrub to catch generic AI-writing patterns the voice rules don't cover.

## The Problem

A voice guide (`Reference/dannys-voice.md` or equivalent) captures what makes YOUR writing sound like YOU — specific word choices, punctuation habits, openers and closers, tone calibration. But voice guides are additive: they push text toward your style. They don't catch generic AI-isms that happen to be style-compliant. A draft can pass every rule in the voice guide and still contain "serves as a testament to," "nestled within the breathtaking," or relentless rules of three.

The fix is two passes, in order, with different rules:

- **Voice pass (positive)** — "this should sound like me." Enforces *your* style.
- **Humanizer pass (negative)** — "this shouldn't sound like any AI." Removes *generic* AI patterns.

Voice first, humanizer second. Running humanizer first would strip the AI baseline but leave the text in a neutral-AI-avoidance state; the voice pass would then layer your style on top. Running voice first anchors the output in your style; humanizer scrubs the remaining AI residue without fighting your voice.

## The Pattern

1. Install the `humanizer` skill (Wikipedia's "Signs of AI writing" guide as a Claude Code skill). The author's install: `~/.claude/skills/humanizer/`.
2. In your vault / project `CLAUDE.md`, add a Voice section:

```markdown
## Voice

Before drafting anything external-facing, read `Reference/your-voice.md` first.

After the voice pass, run the `humanizer` skill as a second scrub to catch generic AI-writing patterns (em-dash overuse, rule-of-three, inflated symbolism, "serves as," etc.) that voice rules don't cover. Voice first, humanizer second, always in that order.
```

3. Write your voice guide (see this repo's `agent-personality-guide.md` for a template).
4. When drafting, both passes run automatically.

## What Humanizer Catches

The skill is ~440 lines covering 24 patterns from the Wikipedia guide. A few examples:

- **Inflated symbolism** — "marking a pivotal moment in the landscape of..."
- **Promotional language** — "nestled within the breathtaking backdrop..."
- **Vague attributions** — "Experts believe..."
- **Em-dash overuse** — three+ em dashes in a short paragraph
- **Rule of three** — forcing every list into three items, even when two or four fit better
- **Copula avoidance** — "serves as" instead of "is"
- **Negative parallelisms** — "It's not just X, it's Y"
- **Synonym cycling** — "However... Nevertheless... That said... Yet..."
- **AI vocabulary** — "additionally," "testament to," "landscape," "ecosystem"
- **Chatbot artifacts** — "Great question!" "I hope this helps!"

## Example Before/After

**Voice-compliant but AI-inflected draft:**

> Additionally, this approach serves as a testament to the importance of thoughtful design. It's not just about aesthetics — it's about meaning. Nestled within every decision is a commitment to quality, simplicity, and clarity.

**After humanizer:**

> This approach shows what thoughtful design looks like. It's about meaning, not just aesthetics. Every decision reflects a commitment to quality and clarity.

Humanizer removed "additionally," "serves as a testament to," "it's not just X, it's Y," "nestled within," and the rule-of-three list ("quality, simplicity, and clarity" → "quality and clarity"). The voice stayed intact.

## When to Use

- Any external-facing text: emails, Slack messages, blog posts, documentation intros, course syllabi, client proposals.
- Drafts that a voice-pass alone can't save — the pattern was originally in the participant's `writing-style` skill as a nested sub-pass for exactly this gap.

## When NOT to Use

- Internal notes, quick Slack replies, or code comments where the voice doesn't matter.
- Text where "AI patterns" are actually load-bearing (e.g., quoting an AI, writing about LLM output).
- Code files. Humanizer is for prose.

## Install

Copy `skills/outreach/humanizer/` to `~/.claude/skills/humanizer/`. Three files: `SKILL.md`, `README.md`, `_meta.json`. The `_meta.json` is harmless metadata for a skill hub (the participant's skill distribution system) and can stay.

## Adjacent Patterns

- Pairs with a voice guide file (`agent-personality-guide.md` teaches this).
- Belongs in CLAUDE.md under a Voice section so every drafting session picks it up.
