---
type: pattern
date: "2026-10-07"
source: A 3D web app moved every visible string, colour, font and decorative prop into theme files, and two full rewrites of its tone the same day touched no code
tags:
  - architecture
  - theming
  - content
  - testing
---

# Every Display String Is Data

Every word a user reads, every colour and font, and every decorative object belongs in a data file the code loads, not in the code. Then a change to how the app sounds or looks is an edit to data, and an agent can make it without touching logic.

## The Problem

A 3D catalogue app had grown the usual way. Signs, the heads-up display and overlay text were string literals in the renderer. Colours and fonts were hard-coded in CSS. Decorative props were placed by hand in scene code. Changing the tone meant a code change in a dozen files, each one a chance to break something unrelated.

## The Pattern

1. **Every display string goes in a copy block.** The theme file has a `copy` object, keyed by purpose (`hud.visits`, `sign.entrance`, `overlay.close`). The code looks up a key and never holds the words. In this app that came to 76 keys
2. **Every colour and font becomes a variable.** The CSS uses custom properties. The loader reads the theme and sets them at startup. No literal colour is left in a stylesheet
3. **Sections override a full default.** One default theme defines every key. Each section (a wing of the building, here) has a partial override that merges over it. A section never has to repeat what it does not change
4. **Decorative props are a list.** The theme holds an array of props with a type and a position. The renderer iterates it and draws each type. Adding a bench is a line of data
5. **A test holds it together.** It collects every copy key referenced in the source and asserts each one exists in every theme. It also asserts that no string in any theme contains a character the house style bans, here the em dash

## The Payoff

The same day the theme system landed, the owner asked for the whole app's tone to change. First shorter and stranger. Then, after reading it, different again: no ego, wonder instead. Both rewrites were edits to JSON. No code changed, the test passed both times, and each rewrite took minutes because the agent doing it could see all 76 strings in one file with their keys beside them.

That is the real gain. Tone is something people decide by reading the result, and they will change their minds. If each change is a code change, each one costs a review. If it is data, they can iterate as often as they like.

## Why It Works

Strings in code are scattered by where they are used. Strings in data are gathered by what they are. Gathered, they can be read as one piece of writing, which is how a reader meets them. The test turns a convention into a check, so a new feature that adds a literal or forgets a key fails right away.

## Watch-outs

- Keys named by place (`line_3`) break the first time the layout moves. Name them by purpose
- Strings that need numbers or names need a small template syntax. Keep it to plain placeholders, or the copy block becomes a second codebase
- A full default is not optional. A partial default means some section will one day render a key path instead of a word

## Adjacent Patterns

- [[grammar-parsing-over-text-matching]] is the same instinct applied to input: give structure a real home instead of matching strings in place
- [[tests-pin-substance-not-identifiers]] is why the test checks that keys resolve and the banned character is absent, not what any particular string says

## Source

A 3D catalogue web app, 2026-10-07. The theme system moved 76 strings, every colour and font, per-section overrides and the prop list into theme files. Two tone rewrites followed the same day and changed only JSON.
