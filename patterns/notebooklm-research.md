---
type: pattern
date: "2026-05-01"
source: Jacob-bd/notebooklm-mcp-cli (primary; CLI + MCP, ~4k stars). Alternatives evaluated and dropped: khengyun/notebooklm-mcp (smaller feature set), PleasePrompto/notebooklm-skill (a Claude Code skill, kept as fallback). Adopted into a personal vault stack 2026-04-22 after a course participant demoed manual NotebookLM use in a diligence workflow, then sharpened during a content-engine vault scaffold on 2026-05-01. Extended 2026-07-25 to always-on agents on a home server; tool landscape re-checked the same day
tags:
  - skills
  - mcp
  - research
  - tokens
  - hallucination-control
---

# NotebookLM as Research Layer

Offload long-document Q&A to Google NotebookLM (Gemini 2.5 over your uploaded sources) instead of feeding the documents into Claude. Only the synthesized answer comes back. Big token saver. Source-grounded answers with citations. Hallucinations close to zero because Gemini answers from the documents, not from training data.

## The Problem

When Claude needs to answer a question that depends on understanding many source documents — a curriculum's worth of session transcripts, a quarter's worth of meeting notes, a stack of API references, a handful of long PDFs — the naive approach is to read all of them into context. Three failure modes:

- **Token cost.** Reading 12 long transcripts plus 12 decks plus 10 reference guides through Claude burns context and money on every question. The same documents get re-read for every adjacent question
- **Retrieval gaps.** Even with grep-style search, single-keyword retrieval misses concepts that span paragraphs or restate ideas in different words. Claude fills the gap with plausible-sounding inventions
- **Hallucinations.** When Claude can't find what was asked, the polite-completion bias produces answers that sound right and aren't

Local RAG mitigates the cost but adds setup hours (embeddings, chunking, retrieval tuning) and still has retrieval gaps.

## The Pattern

Upload your source documents to a Google NotebookLM notebook. Connect Claude to it via an MCP server (or skill, depending on tool form). When Claude needs an answer that depends on the sources, it queries NotebookLM. NotebookLM returns Gemini's answer with inline citations to the specific source documents and locations. Claude uses the answer.

```
Claude has a question
        ↓
MCP tool sends question to NotebookLM
        ↓
Gemini reads the uploaded sources, synthesizes an answer with citations
        ↓
MCP tool returns answer + citations to Claude
        ↓
Claude uses the answer, optionally asks follow-ups
```

A good MCP exposes follow-up tooling so Claude can drill down on the same notebook without re-uploading or re-authenticating each time.

## Why It Works

- **Pre-processing once, query many.** Upload time is a one-shot cost. After that every question is fast and cheap. Compare reading 12 transcripts every time vs uploading them once and querying them 50 times
- **Source-grounded.** Gemini's answer comes from the documents you provided, not its training data. The citation panel shows exactly which source paragraph the answer leans on. You can verify
- **Synthesis, not retrieval.** NotebookLM's Q&A is a real reading-comprehension layer over the corpus, not a keyword search. Concepts that span sources get connected
- **Token shape change.** A 100-page corpus uploaded to NotebookLM costs you nothing in Claude tokens. The same corpus pasted into Claude costs you 100 pages of context every question. Even a few queries pay back the upload effort many times over

## When to Use

- Synthesis questions across many documents — "what topics came up most often in retros," "where did we land on the Path A vs Path B framing," "which student questions appeared in three or more sessions"
- Research over external sources — upload a stack of papers, blog posts, or vendor docs and ask Claude to compare or summarize without burning Claude tokens
- Source-grounded fact retrieval — when the answer must come from a specific document and you want the citation
- Recurring queries against a stable corpus — once the notebook is set up, every question is cheap

## When NOT to Use

- Single-file reads or grep-style searches — read the file directly, faster
- Questions about code structure inside a repo — Claude reads code well, no NotebookLM needed
- Tasks that need to modify the source documents — NotebookLM is read-only by design
- Questions where the answer must come from current web pages — NotebookLM only knows your uploaded sources
- During tasks that don't touch any notebook content — keep the MCP toggled off so its 35 tools don't eat your context window. In Claude Code, toggle with `@notebooklm-mcp`
- **Any source carrying `sensitivity: restricted`** (per `sensitivity-tiered-access-control.md`). Uploading a source to NotebookLM uploads it to Google. The whole point of the `restricted` tier is that the content does not cross a network boundary without explicit per-task authorization, and even then prefers a local model (`local-model-routing-for-restricted.md`). NotebookLM is for `public` / `internal` / `confidential` corpora only

## Tool choice

There is no official Google or Anthropic NotebookLM MCP. Three community options matter, in order of recommendation:

### Primary: `jacob-bd/notebooklm-mcp-cli` (~4k stars)

Unified package. One install, two surfaces:

- `nlm` — command-line interface for scripting (notebook create, source add via URL or Drive or file, audio podcast generation, slide revision, share management, batch ops, cross-notebook queries, pipelines)
- `notebooklm-mcp` — MCP server with 35 tools the AI assistant can call directly

`nlm setup add claude-code` auto-configures Claude Code to talk to the MCP. Same flow for Gemini, Cursor, Cline, Antigravity. Profile management lets you switch between Google accounts (`nlm login --profile work`) when the free-tier 50-queries-per-day rate limit bites.

The CLI is the unlock. With the alternatives you upload sources to the NotebookLM web UI by hand. With `nlm source add --url ...` you script the upload, which matters when you have dozens of transcripts or guides to push.

### Alternative: `khengyun/notebooklm-mcp` (~83 stars)

MCP-only. FastMCP v2, UV install (`uv add notebooklm-mcp`), per-config Chrome profile, init flow that takes one notebook URL. Production-shaped framing (Docker, monitoring) but smaller feature set than jacob-bd. Reasonable if you want a single-notebook MCP server without the CLI surface.

### Fallback: `PleasePrompto/notebooklm-skill` (Claude Code skill form)

Browser automation packaged as a Claude Code skill, not an MCP. Simplest install (clone into `~/.claude/skills/notebooklm/`). Library management with description and topic tags. No CLI surface. Stays useful as a lightweight fallback when the MCP isn't connected, or for ad-hoc queries outside an active project.

## Install (jacob-bd, primary)

```bash
# Preferred: uv tool (gets the latest, isolates the install)
uv tool install notebooklm-mcp-cli

# Or: pipx
pipx install notebooklm-mcp-cli

# Or: pip
pip install notebooklm-mcp-cli
```

After install you have both `nlm` (CLI) and `notebooklm-mcp` (MCP server).

One-time auth (browser-based):

```bash
nlm login
```

A browser window opens, you log in to Google, cookies are extracted automatically and persisted. Future runs use the saved session.

Wire the MCP into Claude Code:

```bash
nlm setup add claude-code
```

This edits Claude Code's MCP configuration so the `notebooklm-mcp` server is registered. Restart Claude Code (or `/mcp` to reconnect) and the 35 NotebookLM tools become available.

Verify:

```bash
nlm doctor
```

## Wiring into the vault

Per the wire-into-existing-flows pattern, document NotebookLM usage in the vault's CLAUDE.md so every Claude session picks it up. Specifically:

- Name the standing notebook for that vault and what it contains
- Note when to query NotebookLM vs read files directly
- Flag the rate limit so it doesn't surprise you mid-task
- Flag the MCP toggle (`@notebooklm-mcp` to disable when not researching) so the 35 tools don't eat context

Without that wiring the MCP exists and doesn't fire on a normal day. The vault's CLAUDE.md is the forcing function.

## Operationalize: bake queries into agent scenarios

A CLAUDE.md mention of "NotebookLM is available" is necessary but not sufficient. The next step is to bake the query into specific agent personas at named scenarios — turn "NotebookLM exists" into "this agent runs NotebookLM at this point in their workflow, with these query patterns."

The shape that proved load-bearing in the course content engine vault was *one agent + one trigger scenario + 1–3 specific query templates,* encoded as a principle in each agent's persona file.

Example, for a content-engine vault with three agents (analyst, PM, drafter):

| Agent | Trigger | Query patterns |
|---|---|---|
| Analyst extracting one source | After section-by-section extraction, before writing summary | (a) "Across all sources OTHER than [filename], what mentions [concept]? Source title + 1–2 sentence context + verbatim quote." (b) "Are there student questions or confusions about [concept] in retros, office hours, or other sessions? Quote with attribution." |
| PM defining a Blueprint slot | When proposing a new article slot | (a) "Rank sources by depth of coverage of [concept]. Top 5 with rationale." (b) "Most-asked or most-debated question about [concept] across all sessions and retros, quoted with attribution." |
| Drafter starting an article | Before writing prose | (a) "5–7 strongest passages about [concept] from across all sources, full attribution." (b) "3–5 video clip candidates with timestamps." (c) "Student stories or before/after moments illustrating [concept] in action." |

Three observations from running this in production:

- **Empty queries are also signal.** When NotebookLM returns nothing useful, the agent logs that explicitly ("CROSS-SOURCE GAP: no other sources cover X") rather than silently omitting the section. The negative result is what tells you the article needs to fill the gap from outside the corpus.
- **The trigger is more important than the prompt.** "Run query X before drafting" only fires if "before drafting" is a checkpoint the agent hits. Encode the trigger explicitly in the persona's principles — not "NotebookLM is available" but "before drafting any prose, run the three queries above."
- **One-shot CLI for synthesis, MCP for follow-ups.** When an agent has a single big synthesis question with no follow-up, the `nlm` CLI is more reliable than the MCP (the MCP defaults to a 120s timeout that bites on cross-source synthesis; verified empirically). Reach for the MCP when the agent will iterate (drill down, ask clarifying follow-ups). Reach for the CLI for one-shot. Either way, MCP-toggled-off when not actively querying.

Example uses for a content-engine-shape vault:

- "Across all Cohort 2 transcripts, which concepts have a clarity-rating of 5 according to the extraction sheets we've already produced?"
- "Find every quote from any retro that mentions onboarding agents. Return verbatim with timestamps"
- "What are the three most-confused-about topics in office-hours transcripts? Cite specific exchanges"

## Extending it to an always-on agent (not a Claude Code session)

Everything above assumes the consumer is a developer in a Claude Code session: a human who knows the notebook exists, types "check my NotebookLM", and sits at a desktop browser when auth expires. Wiring the same layer into a *persistent household agent* (the household agent on a Raspberry Pi, reached over Telegram) broke three of those assumptions at once. Observed 2026-07-25, one evening, one notebook.

### The failure mode of an unwired agent is not silence

The pattern already warns that without the CLAUDE.md wire "the MCP exists and doesn't fire on a normal day." That undersells it. An agent missing a capability does not go quiet and report the gap — **it improvises a worse substitute and reports success, and then defends the gap as a law of nature.** Both halves in one evening:

- **15:13** — handed a shared NotebookLM artifact link, the agent browser-scraped the JS page for the hidden media URL, hand-wrote a mappings file, patched two shared pipeline scripts mid-conversation, spent 137 tool calls and 38 messages in 120 seconds (tripping a rapid-message watchdog, and contributing to a budget alert that evening), then closed by handing the user a shell command to paste. Reported as *"I've cracked this case wide open."* It had, sort of. It also silently edited production scripts on the live host to get there.
- **21:09** — asked to read the notebook itself, it attempted a Google login from a headless browser, hit *"This browser or app may not be secure"*, and told the user his notebook was **permanently gated** and his only path forward was manually publishing public artifact links, forever. The tool that reads notebooks without a browser was one `uv tool install` away. Ten seconds after installing it, the notebook answered.

The second is the expensive one. An improvised substitute wastes a turn; **a confident negative claim closes the door on the real fix and gets adopted as policy** — the agent had already written "root notebooks are blocked by Google login, always require the public artifact link" into a shared skill doc, where it would have taught every future session the same wrong thing. Negative capability claims from an agent are the ones to distrust; see `stale-pointer-asserts-confidently.md`. The corrective is a rule with the incident attached: *when a tool for a job isn't installed, the answer is "let's install it", never "that's impossible."*

### Auth is per-host, and it is the entire problem

`uv tool install notebooklm-mcp-cli` on an arm64 Pi is 30 seconds and works. The install is not the work. **`nlm login` assumes a human at a desktop browser** — a fine assumption on a laptop, an impossible one on a headless host, because Google actively blocks automated-browser sign-in. But `nlm` only needs the browser *at login*; queries afterward run off a saved session with no browser at all. So the shape is: **authenticate where a human and a real browser are, copy the session to where the agent is.**

```bash
# on the human's machine, real browser, real login
nlm login --profile <agent>
# then move the saved session to the agent's host
scp -r ~/.notebooklm-mcp-cli/profiles/<agent> ~/.notebooklm-mcp-cli/auth.json <agent-host>:~/.notebooklm-mcp-cli/
# on the agent's host, no browser involved
nlm login --check     # → Authentication valid
```

Two consequences worth designing for. **Sharing a notebook with the agent's account is not access.** Sharing grants *the account*; the agent still needs that account's session cookies on its own host — a genuinely unintuitive gap, and the one the user hit when he shared the notebook with the agent's Gmail and reasonably expected that to be enough. And **whose session you copy is a privacy decision, not a config detail**: Google session cookies are broad account access, not scoped notebook access. Prefer a dedicated agent account over copying the human's personal session, even though the human's already works.

### Route by topic registry, not by the user mentioning the tool

The pattern's trigger guidance — bake the query into named scenarios in a persona file — holds, but a household agent needs it stronger. **The user will never say "NotebookLM."** He says *"what size battery do we need for the fridge"*. The Claude Code framing ("trigger when the user mentions NotebookLM or shares a NotebookLM URL") never fires on a normal day for this consumer.

What works instead is a small registry the agent consults *by topic*: notebook id, prose `consult_when`, and the keywords the user would actually type. One command routes a question to a notebook and returns the grounded answer, so the agent never picks notebook IDs by hand:

```
ask.py "how big a battery for the fridge?"   → routes, queries, grounded answer
ask.py --which "..."                          → routing decision only, no API call, free
ask.py --list                                 → what's registered and when to reach for it
```

Design notes that earned their place:

- **A free dry-run matters.** `--which` costs nothing against the 50/day limit, so "should I consult a notebook?" is a question the agent can afford to ask before every borderline answer. Charging for that check guarantees it gets skipped.
- **"Nothing matched" is a distinct, non-error exit code.** Exit 3 means *answer normally*. Conflating it with failure teaches the agent to treat the whole layer as flaky.
- **Word-boundary matching, not substring.** "solarium" is not "solar". A router that hijacks unrelated questions gets disabled by the user within a day.
- **The auth-failure message must forbid the improvisation by name.** Not "auth failed" but *"tell the user the session needs re-linking; do NOT attempt a browser login — Google blocks it from here."* Absent that, the agent retries the exact thing that already failed. Same shape as `personality-as-discipline.md`.
- **Registration is the adoption step.** A notebook not in the registry does not exist to the agent. That is a feature: it makes "the agent now knows about X" a reviewable, version-controlled one-line diff rather than a hope.

### A notebook generates, it doesn't only answer — and the agent should offer

Everything above treats NotebookLM as a read layer. It is also a *generator*: from the same sources it will build a narrated video overview, a slide deck, an infographic, a two-host audio deep-dive, a written report. An agent wired only to query is using a fraction of the tool, and the user has to know the feature exists to ask for it.

Three things this changes.

**Generation is async and outruns the agent's timeout.** A query returns in seconds; a video overview takes minutes. An agent with a 60-second tool cap will report failure for a job that is working fine — so artifact creation has to detach, poll, and report back over the messaging channel, exactly like any other long-running job the agent launches (`--background --notify`). Identify the new artifact by diffing the artifact-id list before and after creation rather than parsing the create command's output — it survives CLI version differences, which bit immediately: the flag used to read the new id back existed on one installed version and not another.

**Check what exists before generating.** The notebook in question already had sixteen artifacts, several near-duplicates of each other, because the human had been iterating in the web UI. An agent that generates on request without listing first burns quota to hand the user something they already own. `list` before `create`, and offer the existing one.

**The offer is the feature.** The trigger problem from earlier in this section returns in a harder form: the user does not know the tool can make an infographic, so he will never ask for one. What he does instead is *describe a shape*. Encode the mapping from request-shape to artifact:

| The user says | Offer |
|---|---|
| "help me compare these" / "which should I pick" | infographic — a comparison is a picture |
| "explain this to \<person who won't read it\>" | audio or video overview |
| "I'm presenting this" / "walk someone through it" | slide deck |
| a long back-and-forth becoming a document | report or deck, at the end |

And one level up: **when the user keeps circling a topic — links piling up over days, the agent web-searching the same ground twice — that is a notebook waiting to happen, and the agent should say so** and offer to create it with the sources it has already seen. Offer once, in a sentence, don't nag; generation spends the user's quota, so it stays their call.

**Runtime-created notebooks must not be written to the deployed registry file.** If the registry ships from a repo to the agent's host, the agent registering a notebook it just created writes into a file the next deploy overwrites — the registration silently vanishes, and nobody notices until the agent stops consulting a notebook it made. Split it: the repo file is a curated seed, runtime additions go to a state directory that is never deployed, and the read path merges both (runtime wins on id collision). This is the same class as `migration-blinds-readers.md` — state that lives where a routine process clobbers it.

### Retrieval by description — the user does not know artifact ids

Once the agent can generate, the next thing the user asks is *"send me the one we already made."* Not by name, and never by id: **"the most recent video I made about solar and batteries."** That sentence has three separate things to resolve, and each is a different mechanism:

| Fragment | Resolves via |
|---|---|
| "about solar and batteries" | the topic registry → which notebook |
| "video" | a word→artifact-type map → which kind |
| "most recent" | list order → which one |

Two things make this work in practice. **Order matters more than relevance when the user says it does.** If the phrasing contains a recency word, return newest-first and stop — do not let keyword overlap outvote an explicit "most recent," which is exactly what a naive relevance sort does. Otherwise rank by overlap between the user's words and the *prompt the user originally gave the artifact* (the CLI exposes it as `custom_instructions`), which is a surprisingly good description of what the thing is, because the user wrote it. Both fall out of one stable sort — score descending over an already-newest-first list — plus one early return.

**And it must never return a half-built one.** Filter to `completed` before ranking; an in-flight artifact is a link to a spinner.

On delivery: **there is no artifact-level share link.** Every sharing verb the CLI exposes is notebook-scoped — verified against the installed package, not inferred from docs. Making an entire research notebook public to hand over one video is the wrong trade. But no share is needed: the user owns the notebook and is signed in on their phone, so the plain `…/notebook/<id>/artifact/<id>` URL opens for them straight from the chat. Send the link always, attach the actual file when it fits the messaging platform's upload cap (an infographic PNG or slide PDF does; a video overview generally does not). Link as the primitive, file as the bonus — the reverse strands every large artifact.

### A tool's error message is a hypothesis, not a diagnosis

Roughly half the chat queries against this deployment returned `Google rejected the query (error code 13: INTERNAL)`, with the CLI appending its own theory: *"This may indicate account-level restrictions on programmatic access. Try re-authenticating, or use a different account."*

Both halves of that advice are wrong, and expensively so. gRPC 13 is literally "internal server error" — the identical question answers fine seconds later. The suggested fix, re-authenticating, sends a human to a browser login to cure a transient blip. **Retry before believing an error's self-diagnosis**, especially one hedged with "may indicate."

The near-miss is worth recording, because it happened *while writing the section above about not asserting negatives*. Two probes failed on a notebook shared to the agent's account; a third, on a notebook that account owned, succeeded. That is a clean-looking discriminator, and it produced a confident conclusion — "collaborators can't chat programmatically, only owners can" — that got as far as a capability table, a translated error string telling the agent **not** to retry, and a shipped commit message. A fourth probe, run only because the error text changed shape slightly, returned a full cited answer over the shared notebook and killed the whole theory.

Two lessons, and the second is the useful one:

- **A flaky endpoint manufactures fake boundaries.** With a ~50% failure rate, any two-arm comparison has a 1-in-4 chance of producing a perfect, false discriminator. If a capability difference is real, it survives repetition — so repeat *the failing arm* several times before naming a limit, not just once per arm.
- **Encoding a wrong negative into an agent is worse than leaving the gap open.** The conclusion did not stop at a doc; it became `SHARED_NOTEBOOK_NO_CHAT: ...don't retry, don't re-auth` in the agent's code path. Had it shipped, the agent would have confidently told its user a working feature was permanently unavailable, and suppressed the retry that fixes it. Negative claims get *load-bearing* fast — which is exactly why they need a probe budget proportional to how much behavior they will freeze.

### Refresh the tool landscape before you rely on this section

This pattern's tool comparison dated fast — worth re-checking before adopting, and two things had changed materially within three months of it being written:

- **An official API now exists, for enterprise.** The original section's "there is no official Google API" is no longer true: Google Cloud documents Gemini Notebook API (NotebookLM's new name) methods for enterprise — notebook CRUD, source management, audio overview generation, queries, with VPC-SC and CMEK. A consumer API is still not public, only acknowledged as wanted. So the community tools remain correct for personal use, and are now the *wrong* choice inside a Google Cloud enterprise tenant.
- **`teng-lin/notebooklm-py` is the better fit for an unattended agent.** It offers **master-token auth that mints fresh cookies on demand**, explicitly for servers, CI, and remote MCP connectors — which is a real answer to the per-host auth problem above, rather than the copy-the-cookies workaround. It also names OpenClaw alongside Claude Code and Codex as supported agents, ships a bundled skill, and exposes things the web UI does not (batch artifact download, quizzes/flashcards as structured JSON, mind-map hierarchies, PPTX decks, whole conversation histories as notes). If you are wiring an always-on agent today, evaluate it first.

The `jacob-bd` CLI also moved: artifact provenance (`source_ids` on studio status, so a generated deck traces back to the documents it used), `--json` across the studio creation commands, and a ~60s vertical "short" video format. Version skew between two machines running the same tool is a live hazard — a flag present on one and absent on the other broke a create call mid-build here. Prefer identifying results by diffing state over parsing version-dependent output.

### Separate the knowledge path from the media path

One notebook URL shape is a knowledge source, the other is a media file, and they route to entirely different pipelines:

| URL | What it is | Route |
|---|---|---|
| `/notebook/<id>` | the corpus | query it — knowledge layer |
| `/notebook/<id>/artifact/<id>` | a shared audio/video overview | transcription/ingest pipeline |

The agent had conflated them, which is how a *question about a notebook* became a nine-minute media-scraping session. Naming both shapes explicitly in the routing table is what keeps them apart.

## Adjacent Patterns

- **Wire-into-existing-flows** (`wire-into-existing-flows.md`) — adoption is the wire, not the file. The MCP alone is not enough; the vault CLAUDE.md reference is what makes it load-bearing
- **Humanizer-second-pass** (`humanizer-second-pass.md`) — both are research-or-writing aids that get encoded as CLAUDE.md rules. Pattern shape is similar
- **Doc-warning-preamble** (`doc-warning-preamble.md`) — when a notebook's sources go stale, a preamble pattern keeps readers honest about what the notebook still knows
- **Sensitivity-tiered-access-control** (`sensitivity-tiered-access-control.md`) — any source uploaded to NotebookLM crosses the network boundary to Google. The `restricted` tier is incompatible with this pattern; pair the two patterns at vault setup time so it's clear which sources are notebook-eligible
- **Stale-pointer-asserts-confidently** (`stale-pointer-asserts-confidently.md`) — the always-on-agent section's core failure: an agent's *negative* capability claim ("that notebook is permanently gated") is the least trustworthy thing it produces, and the most likely to get written into a doc and adopted as policy
- **Personality-as-discipline** (`personality-as-discipline.md`) — the fix for "agent improvises instead of routing" is a tell with the incident attached in the identity file, not another keyword row
- **Registry-based-monitoring** (`registry-based-monitoring.md`) — same registry-plus-auto-routing shape, applied to health checks instead of knowledge sources

## Source

Primary tool: [github.com/jacob-bd/notebooklm-mcp-cli](https://github.com/jacob-bd/notebooklm-mcp-cli). Alternatives: [github.com/khengyun/notebooklm-mcp](https://github.com/khengyun/notebooklm-mcp), [github.com/PleasePrompto/notebooklm-skill](https://github.com/PleasePrompto/notebooklm-skill).

Cohort-2 adoption thread: NotebookLM as a research layer surfaced during the course Office Hours 2026-04-22. A course participant was using NotebookLM manually for diligence work, dropping data-room contents in and asking questions there. The author named the pattern, the participant adopted same week. Pattern sharpened on 2026-05-01 during the course content engine vault scaffold when the author compared three implementations and selected jacob-bd's CLI+MCP unification as the primary.
