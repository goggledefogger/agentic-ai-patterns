---
type: pattern
date: "2026-07-28"
source: The agent's own repo, a script (_script_spends function), 2026-07-28. A zero-LLM auditor deciding "does this scheduled task wake a hosted, billed model?" substring-scanned Python source for markers like "claude -p", "import dispatch", "anthropic", "genai", "openai", "gemini". False positives on mock fixture data, docstring mentions, and the detector's own marker list matching itself. False negatives on bare `from google import genai` and raw HTTP calls carrying none of those literal strings. Root cause: text matching cannot distinguish a call from a mention.
tags:
  - testing
  - verification
  - detectors
  - parsing
  - safety
---

# Grammar Parsing Over Text Matching for Detectors

A detector designed to find "does this code spend tokens on a hosted API" substring-scanned Python source for markers: "claude -p", "import dispatch", "anthropic", "openai", "gemini", etc. It found five false positives and missed two real spenders. The pattern: text matching cannot distinguish a *call* from a *mention*.

## The Pattern

Text matching collapses three categories that a detector must separate:

1. **Actually invokes the thing** — a live call that spends tokens (`import anthropic; client = Anthropic(...)`)
2. **Merely names the thing** — a comment, docstring, fixture data, variable name, marker list, or stale example that includes the string but does nothing
3. **Invokes a look-alike** — a free/local alternative that is indistinguishable from (1) at the text layer. `from openai import OpenAI; OpenAI(base_url="http://localhost:1234/v1")` drives local LM Studio: the *same* billed library, same import line, differing only in the parsed hostname of one argument

A regex or substring match fires on (1) and (2) indifferently, and walks past (3) entirely. The tell is simultaneous over-firing and blindness — both symptoms of one root cause.

## Why It Happens

Code has structure that text alone cannot see. An `import` statement has syntactic meaning; a marker string in fixture data does not. A function call's argument list reveals its destination; a string literal in a comment reveals nothing. Reading the *structure* instead of the *characters* makes the distinction obvious.

## The Fix

Parse the grammar, not the text. Language-specific examples:

- **Python imports:** Use `ast.parse()` to find actual `import` and `from ... import` nodes, skip comment/docstring/fixture contexts. One 20-line pass over the AST replaces the substring scan
- **URLs:** `urllib.parse.urlsplit()` on extracted strings to judge the hostname. Only `api.openai.com`, `api.anthropic.com`, etc. mean spend; `localhost:1234` does not
- **SDK calls:** AST inspection for `Call` nodes whose function is `Anthropic(...)` or `OpenAI(...)`, again separating the call from a mention in a config comment
- **CLI shell-outs:** `subprocess.call()` or `os.system()` on the argument list, parsed for real command names vs mentions in a test's mock command string

Parsing is stricter AND simpler than text matching, and it catches both the false positives and the misses in the same fix.

## Second-Order Lesson: Over-Firing and Blindness Are the Same Bug

The temptation after a false positive is to tighten the regex: "gemini matcher is too broad, require 'genai' instead." That's wrong frame. Both the false positive (mock fixture "gemini-3.6-flash") and the false negative (bare `from google import genai`) came from the same root — matching the wrong thing. Narrowing the pattern would have silenced the named case while leaving the detector just as blind to the alternate spelling.

The diagnostic: if a detector both over-fires and misses, stop tuning the pattern and change what it reads. Text → Grammar.

## Third-Order Lesson: Narrowing Demands a Positive Control

After fixing a false positive, the reflex is to run the detector again and see silence. Silence proves nothing. A detector that stops firing because you narrowed it is *worse* than the false positive if narrowing also killed the real fires.

Never declare a fix from silence alone. After any fix, re-run against a case that genuinely SHOULD fire — and if your live system doesn't currently trip it, construct the known-bad case rather than concluding health from silence. Test both directions:

- Positive control: code that SHOULD flag, and does
- Negative control: the false positive case, and it no longer does

Only the pair is evidence.

## Fourth-Order Lesson: Don't Record Fake Approvals

After discovering a false positive, the tempting one-line fix is to add the offending task to a whitelist/allowlist as "approved." That buries the lie in ledger: you've recorded approval for something that spends nothing, and polluted the audit trail. Now the detector is equally blind, the code is equally wrong, and you've added fake provenance.

Fix the classifier, not the ledger. The allowlist should hold genuine approvals for genuine spenders, not cover-ups for detector bugs.

## Why It Works

- A parser reasons about structure, making categories (call vs. mention) explicit in code. Text alone has no way to separate them
- Parsing is more correct AND more concise than tuning regex — once you write the 20-line AST pass, the code is done; regex tuning is infinite
- The control/subject pair (re-test the false positive, re-test a known-good case) surfaces both dimensions of the fix, not just "silence looks clean"
- Fixing the classifier leaves the audit trail clean: a real spend gets flagged, an allowlist entry means approval, not camouflage

## When to Use

- Any detector keyed on text in source code (markers, API calls, config values). Always prefer grammar parsing (AST, language-specific parsers) to regex
- Detectors for infrastructure concerns (cost, capability, security perimeter) where false positives and false negatives both carry real cost
- Any time you're fixing a detector and tempted to "tighten the regex" — stop and ask whether the root is the pattern or the source being read

## When NOT to Use

- A detector on a format that has no grammar (plain text logs, CSV without a schema, arbitrary prose). For those, structured input (JSON logs, explicit manifest) is the prior fix
- A high-volume detector where parsing overhead matters and text matching is good enough. Rare in practice — parsing is almost always fast enough and correctness matters more

## Adjacent Patterns

- `verdicts-from-structured-signals.md`, the runtime sibling: the same wrong-layer bug in code that reads logs, API responses, and command output rather than source — fixed with a signal hierarchy (exit codes → machine formats → sentinels → flagged heuristics) instead of an AST
- `verification-needs-a-negative-control.md`, the control/subject pair discipline that proves a fix is real, not just quiet
- `decorative-gate.md`, the sister anti-pattern where a gate returns "passed" without checking, plays the same role as a false-positive detector: hiding the real work under the appearance of control
- `undefined-blank-is-a-decision.md`, when a detector's vocabulary is undefined (what counts as "approved"?), blanks stay ambiguous and the gate governs nothing
- `scheduled-llm-spend-gate.md`, context: this pattern came from fixing the auditor in that gate's allowlist-check logic

## Source

The agent's own repo, `scripts/change_detect.py`, function `_script_spends`, 2026-07-28. The auditor flagged `local-model-grade` (a log-aggregation script with zero network calls) because the literal string "gemini" appeared in MOCK FIXTURE DATA in its own test. Same tool flagged `local-model-discover` on a docstring "Query LM Studio OpenAI-compatible endpoint" without ever checking that the script only touches localhost. Five false positives in total from that one root cause — the other three being a generated path string, another docstring, and the auditor matching its own marker list. Meanwhile, `from google import genai` and `generativelanguage.googleapis.com` HTTP calls were missed entirely — they carried none of the literal markers. Fixed by moving from regex on source text to: `ast.parse()` for import structure, `urllib.parse.urlsplit()` on extracted URLs to judge hostname, and `Call` node inspection for actual SDK invocations. The parse was stricter (caught the misses) and simpler (20 lines vs. infinite tuning) than regex.
