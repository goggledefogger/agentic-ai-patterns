---
type: pattern
date: "2026-10-02"
source: The dashboard's planning button, redesigned from "always open a new chat on the strongest model" to a decision that reads the chat the press lands in
tags:
  - cost
  - model-routing
  - handoff
  - context
---

# A Model Switch Is a Full Re-Read

Switching models in the middle of a conversation looks free. It is not. The prompt cache belongs to the model that built it, so the first turn on the new model reads the entire conversation again, uncached, at the new model's price. On a long chat that one turn can cost more than the work you switched for.

## The Problem

A dashboard had a planning button that always opened a new chat on the most expensive model. The owner wanted it to do the right thing from wherever he pressed it. Switching the current chat to the strong model seemed like the obvious answer.

At 100k tokens of context and 10 dollars a million input tokens, that switch costs over 1 dollar before the strong model plans anything. A build chat can carry several times that.

## The Pattern

One pure decision function reads the chat the press lands in and picks:

| The chat | Do this |
|---|---|
| Empty | Start here, on the strong model |
| Short history | Switch this chat to the strong model and continue |
| Long history (100k tokens or more) | Do not switch. Write a short handoff on the cheap model the chat is already on, then offer one button that opens a fresh chat on the strong model with that handoff |
| Already running | Show its report |

Two rules carry it:

1. **Conversation size decides between switching and handing off.** Below the line, the re-read is cheaper than losing the context. Above it, a fresh chat with a short handoff is cheaper and usually clearer.
2. **Write the handoff on the model you are leaving.** That model already has the conversation cached, so summarizing it costs little. Asking the strong model to write the handoff pays the full re-read anyway, which is the cost you were avoiding.

## Why It Works

Cost per turn is roughly the context sent times the price of the model reading it, and a cache hit is what makes a long context affordable. A model switch throws the cache away on the most expensive side. The handoff moves only what the next model needs, and it is produced where reading is already cheap.

## When to Use

Any product or habit that moves a conversation to a different model: a "think harder" button, an escalation to a stronger tier, a router that upgrades mid-task. Check how long the conversation is before you switch, and set the line from your own prices.

## Adjacent Patterns

- `put-a-thing-where-it-is-read-at-the-frequency-it-changes.md` is how to write the handoff once you decide to hand off
- `the-conversation-is-unswept-state.md` is what the handoff must not drop
- `delegation-decision.md` picks the unit of work. This picks whether a running conversation should change models at all

## Source

A member's second-brain dashboard, 2026-10-02. The planning button became one decision function with four outcomes, and the 100k line came from the switch cost at the strong model's input price.
