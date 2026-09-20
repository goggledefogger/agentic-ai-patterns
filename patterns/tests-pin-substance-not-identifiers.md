---
type: pattern
date: "2026-07-21"
source: The agent's ops-playbook decomposition (rule 34 relocation, a script), 2026-07-21 review
tags:
  - code
  - testing
---

# Tests Pin Substance, Not Identifiers

A refactor moved rules out of an always-loaded doc into per-topic homes, and a drift test verified each relocated home "contains the rule" by checking two keywords: the topic name and the rule *number*. Then a cleanup pass added a pointer comment — "(rule 34 lived here)" — to one home, and the test went green **without the rule's content ever moving in**. The identifier satisfied the check; the substance was elsewhere (in that case it happened to pre-exist in different words — but the test couldn't tell, and wouldn't have caught it if it hadn't).

An identifier (a number, a slug, a filename) is exactly what a *reference to* the content also contains. Any test keyed on identifiers passes on pointers, tombstones, changelogs, and TODO comments — every artifact that mentions the thing without being the thing.

## The Pattern

When a test must prove content lives somewhere (relocation tests, docs-coverage tests, config-propagation tests):

- **Assert distinctive phrases from the content's substance** — the clause that carries the rule's meaning ("cheapest model that clears", "gate it at creation") — not its name or number.
- Pick phrases a summarizer or pointer would not reproduce: the operative instruction, not the title.
- One phrase per load-bearing clause. If the rule has two teeth, pin both; a single keyword lets half the rule vanish.
- Leave a comment in the test naming the failure mode, so the next person doesn't "simplify" the phrases back to identifiers:

```python
# Keywords must be rule SUBSTANCE, never rule numbers: a bare pointer
# comment ("rule N lived here") satisfies a number keyword and masks
# a rule whose content never actually moved in.
```

Cost of the stronger form: the test breaks when the phrase is legitimately reworded. That's the correct trade — a relocation test that survives a reword it didn't witness wasn't testing anything.
