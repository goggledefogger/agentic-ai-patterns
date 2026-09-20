---
type: pattern
date: "2026-08-11"
source: The personal agent — a vendor billing dispute where a web-search synthesis turned a flat "not supported" into "depends on your specific use case", and the same question answered from the fetched page found four verbatim statements the search never surfaced
tags:
  - research
  - verification
  - tools
  - anti-pattern
---

# A Summary Can Invert the Policy

Search tools that answer in prose are optimizing for a readable middle. That is the right default for "how does X work" and the wrong one for "what does this document say," because policy language is load-bearing at exactly the places a summary smooths: the absolutes, the carve-outs, and the contradictions between two sections of the same page.

The failure is not that the summary is vague. It is that **a hedge and a prohibition are different claims**, and turning one into the other reverses the answer while sounding more reasonable than the source.

## The incident

The question was whether a vendor's prepaid balance could be refunded, ahead of a message that would tell the vendor's own support rep they were wrong.

A web search returned a fluent paragraph: *"support for switching depends on your specific use case and whether you meet eligibility criteria."* Reasonable, cautious, and not what the page says. The page's FAQ reads: *"No, switching from a Prepay billing plan to a Postpay billing plan is not supported."*

Fetching the same page and reading it whole produced three things the search had not:

- The flat prohibition, verbatim, alongside two *other* sections of the same page describing the switch as available to eligible accounts. The document genuinely contradicts itself, which is a materially different finding from "it depends" — one is a fact about the vendor's docs you can quote at them, the other is a shrug.
- Four separate statements of a credit-expiry rule the summary mentioned once and the vendor's support rep had denied outright.
- Two facts nobody asked for that changed the recommendation: prepaid credit was locked to one service, and every other service billed through an uncapped channel. Neither was reachable by a query, because you cannot search for the thing you do not know to ask.

The cost of the fetch over the search was roughly 8k tokens.

## The Pattern

1. **Route by question type, not by habit.** "What does this say" and "is this allowed" are quote-retrieval questions — fetch and read. "How do people usually do X" is a synthesis question, and search is correct there.

2. **When the answer will be quoted at someone, quote the source.** A paraphrase you cannot attribute is not evidence. If the output of the research is a claim in someone else's inbox, the standard is a sentence you can point at.

3. **Read the whole page once, not the answer to your query.** The findings that change a recommendation are usually adjacent to the question rather than inside it. A targeted extraction returns what you asked; a full read returns what you needed.

4. **Treat self-contradiction as a finding, not noise.** Documents that contradict themselves are common in policy and pricing. A summarizer resolves the conflict silently, and the resolution it picks is unattributable. Surfacing both sentences is more useful than either one.

5. **Price it honestly and pay it selectively.** A full fetch costs meaningfully more context than a search snippet. That is worth it when someone will act on the answer, and wasteful for orientation. The tell that you are in the expensive case: you are about to write the claim down somewhere it will be read by a third party.

## Watch-outs

- **A fluent hedge reads as careful.** "Depends on your specific use case" pattern-matches to good epistemic hygiene, so it survives review that a bare wrong answer would not.
- **Confidence travels; the source does not.** Once a paraphrase enters your notes it loses its provenance, and the next reader cannot tell a quote from a gloss. Keep the verbatim sentence attached to the claim.
- **The search result and the page can both be current and still disagree** — the summarizer is compressing, not stale. Do not diagnose this as a freshness problem; see `freshness-axis-must-match-the-question` for the case where it genuinely is one.
- **Two sources beat one, and independence is what makes them two.** Two pages from the same vendor stating the same rule is a much stronger position than one, and it closes off "that documentation covers a different product."

## When NOT to use

Orientation, prior-art scans, and "what are the options" questions are what search synthesis is for, and fetching every candidate page to answer them is how a cheap survey becomes an expensive one. The pattern applies when the exact words carry the decision.

## Adjacent Patterns

- `a-source-assertion-pins-your-belief-not-the-behavior` — the same gap between what a thing *says* and what is *true*, one layer down in code
- `a-fast-answer-is-a-suspect-answer` — the cost/confidence tradeoff stated generally
- `a-partial-read-proves-presence-not-absence` — why the full read, not the targeted extraction
- `freshness-axis-must-match-the-question` — the neighbouring failure this is often mistaken for
- `contradiction-surfacing` — flag the conflict, never resolve it into confident prose

## Source

The personal agent session 2026-08-11, verifying a support rep's claim before a reply went out. The rep's assertion, the search summary, and the vendor's own documentation were three different answers; only the last one was quotable, and it was the one that turned out to be right.
