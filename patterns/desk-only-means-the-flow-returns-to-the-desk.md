---
type: pattern
date: "2026-09-19"
source: The dashboard mobile pass, 2026-09-19 — a phone on the member's tailnet could read the brain but not set up an engine or connect a key, and the owner asked "does install/sign in need to happen desk only?"
tags:
  - security
  - mobile
  - remote-access
  - product
---

# Desk-Only Means the Flow Returns to the Desk

A local web app with no login was reachable from the member's phone over a private network. The first security pass sorted its routes into "reads, open" and "writes, desk only", which is the right first cut: a lost phone must not be able to turn approvals off. But the cut was made by caution, so it swept in every write, and the new-member flow on a phone died at the first step that mattered: entering a provider key, fetching an engine, signing in. The owner's question was the right one: does this step *need* the desk, or is it just safer there?

## The Pattern

- **Gate by what the step physically needs, not by how it feels.** A step is desk-only only when its flow returns to the desk: an OAuth callback that lands on the machine's own localhost port, a sign-in that needs typing into a terminal on that machine, or a click that is itself the approval and must happen where the agent cannot reach. Everything else is the member acting on their own brain from wherever they are.
- **Sort by action and by method, not by route.** One route can carry a read-only check, a catalog install and three kinds of sign-in. A key is text and travels. A device-code or link-and-code flow travels, because the link opens wherever the member is and the code pastes back from anywhere. A callback to localhost does not. Write the predicate once, next to the desk-only list, and let it ride out to the page so the page never re-derives the rule.
- **Say it before the tap.** A step that stays desk-only is shown on the phone, dimmed, with the reason in the app's own words ("this one signs in through your computer"). A refusal after the tap is a bug report waiting to happen; a refusal that a re-render then wipes is worse.
- **Read the method classes from the engine, not from docs.** What each engine actually advertises (its handshake, its auth methods, whether a key variable exists) decides the class. When no installed engine offers a device-code method, link-and-code is a follow-up, named, not a class the code pretends to support.
- **Keep the real gates.** Approvals, the safety switch, identity, restart, update: the click is the point, and it happens at the desk.

## Why It Works

- The caution-shaped gate protects against the wrong thing. The dangerous writes are the ones that change what the agent may do; a member's own key or a catalog engine changes what pays or what is available. Sorting by consequence keeps the safety property and returns the phone to the member.
- A predicate the page receives cannot drift from the one the server enforces, so the dimmed rows and the 403s always agree.

## When to Use

- Any no-login local app that becomes reachable from a second device (a tailnet, a LAN, a tunnel).
- The moment a member asks "does X need the desk?", answer with the flow's mechanics, not with a policy. If the honest answer is "no", the gate was caution.

## Adjacent Patterns

- `settings-follow-the-member-not-the-browser.md` — the same question for state: what is about the member versus about the device.
- `the-agents-home-is-not-its-library.md` — split the variable when a member asks the same thing three times.
