---
type: pattern
date: "2026-06-16"
source: A course content-engine portal dogfood (2026-06-16) — getting opencode's agentic tier to execute tools with a local model on a 16GB M1 Pro. Empirically validated (rung-3 probe PASS), not aspirational.
status: validated
tags:
  - local-models
  - agentic
  - tool-calling
  - opencode
  - llm-policy
---

# Local-Model Agentic Tool-Calling

"The model runs locally" is not "the model can drive an agentic tool loop." A local model wired into an agentic runner (opencode, Aider, Claude-Code-style harnesses) can *describe* `Read`/`Write`/`Edit`/`Bash` as prose and never emit a real tool call — so the run produces no files, no commit, no effect, while looking like it worked. This pattern is the discipline for closing that gap: classify capability, fix the non-obvious config traps, and prove execution end-to-end before trusting it.

Companion to [`local-model-routing-for-restricted.md`](local-model-routing-for-restricted.md) (that one is about *which* tier goes local for privacy; this one is about *making the local agentic path actually work*).

## Emit vs Execute — the distinction that bites

There are two separate questions, and passing the first does not answer the second:

1. **Does the model EMIT structured tool-calls?** A small isolated prompt with one tool either returns a real `tool_calls` object or writes `{"name":"write",...}` as text content. Necessary.
2. **Does the runner EXECUTE them end-to-end?** Drive the real runner on a trivial task (write a file, edit it, git-commit it) and check the **file and commit actually landed on disk**. Sufficient. This is the only bar that matters.

The realistic failure is silent: a weak model emits tool-calls as text, the runner "describes" the work, and the operator sees a plausible transcript ending in "DONE" with nothing on disk.

## The traps (each cost real time)

### 1. The agentic-prompt context trap (the big one)

An agentic runner's system + tools prompt is large — opencode's *base* is **~23K tokens** (`n_keep ≈ 22803`), and a **real step** (step template + injected source/context) is bigger still: a portal "express" run measured **~36K** (`n_keep 35714`). So a trivial probe under-counts what real work needs. Load the model with context **≥ ~48K (64K is a safe default)** — 32K passes a toy probe yet overflows on a real step. The two local runtimes fail *differently* when it's too small:

- **LM Studio errors loudly** on overflow (`n_keep ... >= n_ctx: 8192`). Its JIT auto-load uses the model's *default* context (often 8K) — so a first request can fail this way. Pre-load big: `lms load <model> --context-length 65536`.
- **Ollama silently truncates** to its 4K default — *no error*. The model never sees the front of the prompt where the tool-use instructions live, so it falls back to emitting tool-calls-as-text. This looks identical to a capability failure but is pure config. Fix with a real Modelfile tag: `FROM <model>` + `PARAMETER num_ctx 65536`, then `ollama create <m>-64k -f Modelfile`.

Consequence: a model that passes a *small-prompt* emit-probe can still fail in the real loop purely from truncation. Never conclude from the small probe alone.

### 2. The runner's model list is a stale catalog

opencode's `opencode models` lists models from a catalog (models.dev), **not** your installed set — it shows models you don't have and omits ones you do. Referencing an unregistered id throws `ProviderModelNotFoundError`. Register the model explicitly (for opencode: `~/.config/opencode/opencode.json`, `provider.<id>.models` with `tools: true`). And the catalog's `:7b-32k`-style context variants are not real Ollama tags — the runner passes the string literally and Ollama 404s. Conversely, that `tools: true` you write is a *claim*, not a measurement: qwen2.5-coder carries `tools: true` and still emits tool-calls-as-text. So if you auto-build a model menu *from* `opencode.json` (nice: it reflects the operator's actually-registered models instead of a static list), don't treat every `tools:true` entry as agentic-ready — gate on the empirical end-to-end probe, or carry a per-model caveat. (the course portal sources only `provider.lmstudio.models` and tags the weaker one.)

### 3. Model family matters more than "coder" in the name

**Everything in this section is a 2026-06-16 snapshot of a fast-moving field — re-probe before treating it as current.** The durable claim is the *shape*: `tools: true` is a catalog assertion, not a measurement, and family beats naming. The specific verdicts below have a half-life measured in months. Contrary practitioner testimony already exists (2026-08-11: gpt-oss reported as the first open model genuinely reliable at tool calling, and Liquid AI shipping a ~2.6B specialised for it) — unverified here, and named so a reader knows the snapshot is contested rather than settled. Rung-3 probe first, this list second.

At the ~7–9B size that fits a 16GB machine, the **Qwen3 / Qwen3.5** family has the most reliable native tool-calling. The "canonical" local coder **qwen2.5-coder fails** through opencode — it emits tool-calls as a JSON text block even at full context, despite `tools: true`. Gemma/medgemma: no native tool-calling worth relying on. The smallest dedicated `qwen3-coder` is 30B (too big for 16GB) — use `qwen3:8b` / `qwen3.5-9b` instead. (Verified: `qwen3.5-9b` in LM Studio at 64K → rung-3 PASS, and a full express run past the context overflow.)

### 4. Memory discipline on 16GB-class machines

Loading two models at once — or LM Studio and Ollama both holding a model resident — thrashes swap and can lock the machine. One model at a time; unload between tests (`lms unload --all`; `ollama stop <model>`). A 9B at Q4 + 32K KV cache is ~7–8GB resident: fine alone, fatal alongside a second.

**The two-wall ceiling (16GB).** KV scales with context, so there's a hardware squeeze on the heaviest steps. A step's context *grows across the tool-loop* (each turn appends tool outputs), so a web-fetch-heavy step like "research" overflows a 64K window mid-loop — but bumping the model to 128K pins ~12G wired and thrashes swap to ~11G (near-lockout) on 16GB. Wedged: too big for 64K, too heavy for 128K. Verified 2026-06-16: the portal's research step executed real tools at 64K (proving the wiring) then hit "Context size exceeded," and 128K wasn't loadable without swap thrash. Conclusion: on 16GB, reserve local for the probe + lighter agentic steps; route heavy web-research steps to a big-context cloud agent (~200K) or a >16GB machine.

### 5. Probe serially

opencode boots a server subprocess per run and swaps `process.env` globally per boot. Never run two runner boots (probes or app dispatches) concurrently — serialize them.

### 6. Don't edit app source while a run is live

A dev server with HMR (Vite/Astro/Next) reloads the program on any source change — which wipes in-memory run state. Edit a source file mid-run and the in-flight agentic run dies: its `/stream` 404s, the UI shows a half-finished step, and the spawned opencode subprocess is orphaned (still `GENERATING`, detached from the server that can no longer reach it). This is distinct from a *browser* reload, which reattaches fine to a still-live server-side run. If you want runs to survive a server reload, persist run state to disk; otherwise leave source alone while a run is live (a real bite during this dogfood — a doc edit to `config.ts` killed a research step mid-flight).

### 7. Cancel hangs on an idle local stream

A backend that consumes the runner's SSE event stream with `for await` and only checks the abort flag at the *top* of the loop will not observe a cancel while the model is idle — it's blocked awaiting the next event that never arrives, so its teardown `finally` never runs. The run is then stuck `running` after a cancel that visibly stopped the model, *and* it still holds the serial mutex, so the next run deadlocks behind it. Fix: race the abort against `iterator.next()` (a `firstSettled(p, signal)` helper) so a cancel breaks the loop promptly → teardown fires → the run settles to `cancelled` and releases the lock. The well-behaved *fake* runners in your tests won't catch this (they yield in a tight loop, so the top-of-loop check fires immediately) — only a real, idle network stream exposes it. (the course portal: `opencode.ts:firstSettled`, dogfood Finding D.)

### 8. Capability belongs to the (model, runtime) PAIR — and "installed" is not "serving"

Two inventory traps, both measured 2026-08-01 on the 16GB machine, both of which sent a session toward a download it did not need.

**Capability is not a property of the model.** `medgemma` reports `capabilities: ["completion"]` under Ollama and `vision: true` under LM Studio — same family, same machine, same hour, opposite answers. So a capability finding never travels: ask the runtime you will actually dispatch through, and never infer from a model's name, family, or reputation. Both runtimes expose it as a *structured* field, so there is no excuse for parsing prose: `POST :11434/api/show` → `capabilities[]` for Ollama, `lms ls --json` → `vision` / `trainedForToolUse` for LM Studio.

**A model you own does not stop existing when its host process is stopped.** An inventory script that enumerated LM Studio by probing `:1234/v1/models` reported `active: false, models: []` whenever the server was down — which is most of the time, since it starts on demand. Seven installed models, three of them vision-capable, were invisible. `lms ls --json` reads the on-disk index with the server stopped. Keep `installed` and `serving` as separate fields; collapsing them hides the whole library behind a stopped process, and the failure is silent because an empty list is a perfectly plausible answer.

Same shape as trap 2's `tools: true` caveat, one layer down: the *declared* capability and the *measured* one are different facts, and so are the *owned* inventory and the *running* one.

## Gate the agentic path (don't let it default on)

- **Completion tier** (single-shot text) is safe on any model that returns real text — wire it freely.
- **Agentic tier** is selectable ONLY after end-to-end execution is confirmed for the chosen model. Default it OFF; require explicit operator confirmation (an env allow-list the operator sets after the probe passes). Validate the backend at the *dispatch seam*, server-side — not just in the UI — so a raw client request can't pick an unconfirmed local backend. (See the course portal's `capability.ts` gate + `AGENT_AGENTIC_TOOL_CAPABLE` env allow-list for a worked example.)
- **Ship a cloud fallback, surfaced at the failure.** Because local *will* hit the two-wall ceiling on heavy steps, a big-context cloud agent (claude-code) has to be a first-class fallback. Detect the capacity/context error in the run UI and offer a "switch to claude-code" action right where the run failed (then resume the failed step) — don't leave a dead-end error whose only recovery is buried in a settings page. (the course portal: the express run-error triages a local context-overflow to a "switch to claude-code" link; the Settings setup-help also notes the fallback.)

## Route per-step, not all-or-nothing (the two-wall ceiling answer)

The fallback above is run-level — it rescues a wedged run. The structural fix is finer: when local does the *light* steps fine but wedges on a *heavy* one, "all local or all cloud" is the wrong granularity (it pays cloud for steps a local model would do at $0). Two strategies:

1. **Per-step engine routing.** Let each step pick its engine (harness · model) before the run, with a smart computed default: heavy/unbounded steps (web research) → big-context cloud; bounded file-in/file-out steps (SEO, draft, certify) → local. Carry a per-step backend map through dispatch (the seam usually already reads a backend per call). **Readiness then becomes per-step, not global** — a step routed to the cloud agent needs its session capability (e.g. a subscription verify) surfaced *before* the run, not as a 409 mid-dispatch. And keep the two concerns separate: *verifying* a capability grants it session-wide; it must NOT also flip the global default, or the carve-out silently escalates the whole run onto the expensive path. (the course portal: `step-routing.ts` `resolveStepEngine` + `unreadyBackendsForRun`; the per-step dropdown sources its local models from the operator's real `opencode.json`.)
2. **Step decomposition.** The deeper fix for the heavy step itself: the wall is one loop accumulating every fetch into a growing context. Subdivide — fetch+summarize one source per bounded sub-invocation, discard the raw page, then synthesize the summaries — so each invocation stays small and local stays viable end-to-end. (The honest deferred mechanism; per-step routing ships first.)

## The executable form

This pattern is encoded as the `integrating-local-models` skill (`a personal skills repo`): `probe.sh` for the rung-1/2/3 emit check and `probe-opencode.sh` for the sufficient end-to-end execution check (real file + commit verified on disk).

## Adjacent Patterns

- **[`local-model-routing-for-restricted.md`](local-model-routing-for-restricted.md)** — the privacy-routing companion; once the agentic path works locally, it can serve the restricted tier.
- **[`smoke-tests-with-real-data.md`](smoke-tests-with-real-data.md)** — same spirit: prove it against reality, not a mock.
- **[`framework-gotcha-comments.md`](framework-gotcha-comments.md)** — the context-window and stale-catalog traps are exactly the kind of non-obvious constraint that earns a 2-line comment at the config site so a future reviewer doesn't "simplify" it away.
