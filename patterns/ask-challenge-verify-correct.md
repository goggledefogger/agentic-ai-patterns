---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo)
tags:
  - teaching
  - onboarding
  - ai-literacy
---

# Ask-Challenge-Verify-Correct

A four-step framework for teaching non-technical people to use AI tools: ask the system, challenge the output, verify independently, correct if wrong. Builds trust, domain knowledge, and verification habits simultaneously.

## The Problem

Teaching someone to use AI tools has a bad failure mode on each end of the credulity spectrum:

- **Trust-everything** — they accept AI output as fact, get burned once, lose trust permanently.
- **Trust-nothing** — they treat AI output as unreliable garbage, do everything manually, and miss the leverage entirely.

Neither is useful. The useful stance is calibrated trust: trust the AI's speed, verify the AI's claims, know which claims need verification. That calibration is a skill, not a disposition, and it's teachable.

## The Pattern

Four steps, in order, applied to every meaningful task:

### 1. Ask the system

Phrase the real question, not a simplified version. Give context. Let the AI work.

"What's the status of the LP consent process for Acme Fund II?" not "LP consent status?"

### 2. Challenge the output

Don't accept the answer at face value. Ask:
- "How confident are you about this?"
- "What are you assuming?"
- "Where did this information come from?"
- "What's the counter-case?"

If the AI can't answer those, the output isn't ready to use.

### 3. Verify independently

Spot-check the AI's output against a source it didn't use:
- Check the Airtable record directly.
- Read the original email thread.
- Ask the person the AI's answer is about.

Verification is a habit, not a one-time gate. The goal is to build the instinct for *which* claims need verification — usually: anything about people, anything with a number, anything about the recent past, anything involving a commitment.

### 4. Correct if wrong

If the AI was wrong, correct it in a way that persists — not in the chat, where the correction disappears. Options:
- Update the source data so the AI sees the right thing next time.
- Add a note to the agent's skill / SOUL / CLAUDE.md with the correction.
- Write a decision doc if the correction represents a policy.

Correction without persistence is re-work.

## Why It Works

The four steps build three things simultaneously:

- **Trust in the system** — you see it succeed at things you verified, so you trust those categories.
- **Domain knowledge** — you learn *what data exists*, *what's reliable*, *what the AI is guessing about* — all from the verification step.
- **Verification habits** — the instinct for which claims need checking generalizes far beyond AI tools.

The participant's team-onboard skill builds all three at once. A student who runs through a week of ask-challenge-verify-correct with real work knows the CRM, knows the agents, and knows what to trust — not because they were told, but because they verified.

## Example Script (for a Course TA)

```
This week, you're going to use the operations persona (the COO agent) to prep for the
portfolio sync. Here's how to work with it:

1. ASK: Paste the agenda and ask "what should I bring to this meeting?"

2. CHALLENGE: When the operations persona names a portfolio company or a commitment,
   ask it: "Why do you think that's relevant?" and "When did this come up?"

3. VERIFY: Pull up the Granola transcript the operations persona cited. Does the company
   name appear? Did the commitment actually get made?

4. CORRECT: If the operations persona missed something obvious, tell me — we'll add it to
   the skill so next time it's in the default context.
```

## When to Use

- Any team member learning to use an AI tool for real work.
- Course cohorts where students are using AI on their own projects.
- Onboarding an agent to a new domain (you use the same framework on yourself).

## When NOT to Use

- Low-stakes ad-hoc use (asking Claude to summarize an article). The overhead of four steps outweighs the risk.
- Purely generative tasks where "correct vs incorrect" isn't meaningful (brainstorming, creative writing first drafts).

## Adjacent Patterns

- Pairs with milestone-based onboarding: the four steps are applied to *real* work, not to abstract exercises.
- Feeds the decision-doc pattern: corrections that represent policy shifts become decision docs.
