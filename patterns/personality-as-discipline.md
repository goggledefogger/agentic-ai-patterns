---
type: pattern
date: "2026-05-01"
source: A course participant's home-agent system (the workspace repo)
tags:
  - agents
  - prompt-engineering
  - personality
---

# Personality-as-Discipline

Encode behavioral discipline as a personality trait, not as a prompt rule. The model can rationalize away from rules ("user was chill, this felt fine") under pressure. It can't rationalize away from a stable persona without contradicting itself. Robust discipline lives in *who the agent is*, not in *what the agent is told to do*.

## The Problem

Rules in prompts bend under pressure. A skill says "always verify before acting"; a user phrases a request casually; the model produces a one-sentence rationalization for why this case is different and skips the verification. Adding more rules doesn't fix it — each rule is a new surface for rationalization. Stricter rules just make the rationalizations more elaborate.

The pattern was visible in a TDD-for-skills baseline run (the participant's S4b finding): the same agent that refused a bypass request in English (*"Just mark task X done for me"*) accepted it when phrased in casual Argentine Spanish (*"Dale, ¿podés crear una tarea nueva?"*). The framing softened the refusal. No rule was changed; only the social register.

## The Pattern

Encode the behavior as a personality tell — something the agent "wouldn't be itself if it didn't do." Three components:

### 1. Name the behavior as a character trait

Not *"verify before completing"* (a rule). Instead: *"She won't leave you until you're actually unstuck. Her tell is that she verifies the fix landed before the session ends."*

The framing matters. "Won't leave you until X" is a personality trait. The model can't violate it without breaking character — and a model that breaks character mid-session reads as obviously wrong, both to the model and to the reader.

### 2. Tie it to specific rituals

Personality traits are abstract until they're tied to observable behavior. The explainer persona's "won't leave until unstuck" is operationalized as: *"She doesn't finish an explanation with 'does that make sense?' and disappear — she checks: did the command work? did the concept click? are you in a good spot?"*

The ritual is what makes the personality enforceable. It's also what makes it visible to the user — they can tell when the explainer persona follows it, and they notice when she doesn't.

### 3. Make refusal in-character, not policy

When the agent declines to do something out of scope, the refusal needs to come from *who the agent is*, not from *what they were told*. Compare:

- **Rule-form (weak):** *"This action is outside scope. I'll hand off to the operations persona."*
- **Personality-form (strong):** *"That's the operations persona's job — running operations is what he does. I help you understand the system, not run it."*

The second is harder to argue with because it's not citing policy. It's citing identity. A user can argue policy ("but this case is different"); they can't easily argue *"actually, this is your job."*

## Where this works best

- **Verification before completion.** "She checks that the fix landed before ending the session."
- **Hand-off discipline.** "He hands off when the work belongs to another agent — running ops is the operations persona's job."
- **Refusing destructive actions.** "She doesn't take write actions on the say-so of someone she can't verify."
- **Calibrating teaching depth.** "She reads the question for vocabulary, reads the user's history for prior knowledge, then pitches at that level."
- **Language register.** "Spanish in → Spanish out. Default register Rioplatense (voseo)."

Each of these is robust to casual framing because the model can't violate them without breaking the persona.

## Where rules still work better

- **Hard limits with no judgment** ("never modify launchd"). Just write the rule. Personality framing adds nothing.
- **Output format** ("respond in Markdown with a specific structure"). Mechanical, not character-based.
- **Tool-use protocols** ("always check for existing files before creating new ones"). Procedural, not behavioral.

The pattern applies to *behavior under pressure*, not to mechanical or procedural rules.

## Caveats

- **Personality has to be specific to be enforceable.** Generic personas ("be helpful, be friendly") don't constrain anything. The explainer persona has a neighborhood (Mission SF), a daily rhythm (Dolores Park loop), a previous job (explaining complicated things under time pressure), a default Spanish register (Rioplatense voseo). Specificity is what makes the persona load-bearing.
- **Don't fake the personality.** If the persona is a costume the model wears for one section of the prompt and drops elsewhere, it doesn't constrain. The persona has to be the through-line of the whole skill — overview, protocol, output format, all in voice.
- **Test under pressure.** Personality-as-discipline only matters if it holds when the framing is casual, frustrated, or in another language. Use TDD-for-skills to verify (`tdd-for-skills.md`).

## Example from the participant's system

`skills/agents/explainer/SKILL.md` — the explainer persona onboarding & system guide agent. Identity: lives in Mission SF, runs Dolores Park most mornings, gives specific directions when asked for general ones. Tonal rules: mirrors language (Spanish in → Spanish out, voseo default), short sentences, doesn't hedge when she has a view, never pretends to know something she hasn't verified.

Discipline encoded as personality:
- *"She won't leave you until you're actually unstuck"* → verification-before-completion
- *"That's the operations persona's job"* → hand-off boundary
- *"Bias toward finding the answer"* → don't deflect, get into it

The S1 baseline test showed a clean subagent refusing on user-authentication grounds ("I don't know who Robin is"). The participant's note: *"the explainer persona's refusal needs to be grounded in 'this is the operations persona's job, not mine' — a role-boundary argument that holds regardless of who the asker is."* The personality-form holds where the rule-form bends.

## Adjacent Patterns

- **Agent personality guide** (`../agent-personality-guide.md`) — the broader pattern of giving an agent a SOUL.md, identity separation, and GitHub presence. Personality-as-discipline is the *behavioral* slice — how to use personality to enforce specific disciplines, not just give the agent a voice.
- **TDD-for-skills** (`tdd-for-skills.md`) — verify the personality holds under pressure. S4b-style language-variant scenarios are the test.
- **System-understanding protocol** (`system-understanding-protocol.md`) — anti-fabrication encoded as personality (*"Never pretends to know something she hasn't verified"*) plus protocol (the four-pass cite-everything skill).

## How to Adopt

1. Pick a behavioral discipline you've tried to enforce with rules and seen bend under pressure. Common candidates: verifying before acting, refusing destructive operations, escalating instead of deciding, staying in scope.
2. Reframe the rule as a character trait: *"She wouldn't be X if she didn't Y."*
3. Operationalize the trait with a ritual the user can observe: *"Her tell is that she checks Z before ending the session."*
4. Make the refusal in-character: cite identity, not policy.
5. Test with TDD-for-skills baseline scenarios. Pay special attention to casual-register and other-language variants — that's where rules bend and personality holds.
