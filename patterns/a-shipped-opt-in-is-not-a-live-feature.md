---
type: pattern
date: "2026-11-03"
source: The personal agent (2026-11-03): a fingerprint-gated phone unlock shipped on 2026-10-14 behind an opt-in switch, and 3 weeks later the owner asked for the same ability as if it did not exist
tags:
  - verification
  - opt-in
  - drift
  - features
---

# A Shipped Opt-In Is Not a Live Feature

On 2026-10-14 a fingerprint-gated unlock for the phone shipped. With one opt-in switch turned on at the desk, phone chats could reach private stores in a sealed mode with no web access. Three weeks later the owner asked for the phone to do what the desk does, as if nothing existed.

The merge record said done. The live state said otherwise: the switch had never been turned on. Every phone session since had hit the old wall. An earlier chat had assumed the flow worked because its setup step succeeded, and nobody had read the switch.

## The Pattern

Before designing more, read the live state of the switch, not the merge record.

1. **Treat "shipped" and "on" as 2 facts.** A feature behind an opt-in is live only when someone has opted in. The merge says the code exists
2. **Read the switch where it lives.** Open the setting, the flag file, the table row. A successful setup step, a green test, or a closed ticket reports on the setup, not on the state
3. **Put a check or a prompt where the person would want it.** An opt-in nobody is reminded of ships dark. When the owner hits the wall the feature was built to remove, the wall itself can say "this can be unlocked, switch X is off"
4. **Report the gap as the finding.** "Already built, switch off" is a smaller job than a new design, and it is the likeliest true answer when a request repeats something recently shipped

## Why It Works

- **Opt-ins decay into dead code.** Defaults get used, and anything that needs a deliberate step is skipped by the one person who forgot it was needed
- **A request is evidence.** When the owner asks for something that was delivered, either it does not work or it is not on. Both are cheaper to find than to rebuild around
- **The live state is one read.** The merge record can be argued about, a flag is either set or not

## When to Use

Whenever a request sounds like something already built, and when closing out any feature that needs a person to turn it on. Pair it with the checks for the fix itself, since this one only asks whether the thing is on.

## Source

The personal agent, 2026-11-03. See `a-write-up-outran-the-merge` (a documented fix is not a landed fix, one hop earlier than this), `version-bump-is-the-delivery` (merged is not shipped), and `freshness-axis-must-match-the-question`.
