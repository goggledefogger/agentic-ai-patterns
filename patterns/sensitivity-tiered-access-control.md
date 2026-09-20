---
type: pattern
date: "2026-05-05"
source: A work vault's sensitivity-tier decision record, 2026-04-30, plus gaps observed routing a confidential transcript on 2026-05-05. Revised 2026-07-16: the escape hatch it tolerated was closed by native agent-harness controls that did not exist in May, and the filesystem-level approach the doc had rejected as too invasive was vindicated
tags:
  - sensitivity
  - privacy
  - hooks
  - llm-policy
  - layered-defense
---

# Sensitivity-Tiered Access Control for Vaults

A three-layer defense for keeping sensitive content out of demos and out of cloud-AI processing without strangling the agent's productivity on everything else. **Tagging declares the tier, hooks gate the tier, conventions ask the agent to honor the tier — in that order of strength.** The agent's honor system is the weakest layer and the only one that survives if the others fail. Treat it accordingly.

## The Problem

A working Obsidian vault accumulates content of mixed sensitivity. In any given vault, you have:
- Public docs (READMEs, decision records, conventions)
- Internal facts (team people files, project status, contacts)
- Confidential material (annual review packets, candidate notes, 1:1 transcripts)
- Restricted personal/legal content (counseling notes, health context, draft separation conversations)

Most agent-driven workflows want to read freely across the vault — that's how transcript processing, packet drafting, and meeting briefs work. But a small set of files must not flow into cloud-AI processing without explicit per-task authorization, and a slightly larger set must not surface during screen-share demos.

The naive solution is "trust the agent to remember" — usually a CLAUDE.md rule like *"don't read files in `personal/`."* This fails on three predictable axes:
1. **Context loss.** A session compaction or new agent run loses the rule.
2. **Skill drift.** A skill that auto-processes content (`/transcript-processor`, `/incorporate`) doesn't necessarily check folder rules.
3. **Tool diversity.** Bash, MCP servers, and other agents bypass CLAUDE.md entirely.

The fix isn't more rules. It's putting the enforcement at a layer the agent can't forget.

## The Pattern

Three layers, deployed in this order of priority:

### Layer 1 — Tagging (declarative, machine-readable)

Every file declares its tier in YAML frontmatter:

```yaml
---
type: snapshot
sensitivity: restricted
---
```

Four values, semantically distinct:

| Value | Demo OK? | Cloud AI OK? |
|---|---|---|
| `public` | yes | yes |
| `internal` | no | yes |
| `confidential` | no | yes (in private session) |
| `restricted` | no | **no — ask first** |

Why frontmatter, not folder placement: folder placement breaks down for mixed-content files (a team profile with public icebreakers + confidential 1:1 notes). Folder placement also breaks if you reorganize. Frontmatter travels with the file.

Why the same property carries both demo and AI semantics: two separate properties (`sensitivity` + `ai_access`) doubles the schema for marginal precision. A single four-value enum covers the realistic cases.

A `.llmignore` at vault root mirrors the policy in vendor-neutral gitignore syntax. Today it's documentation; tomorrow it's load-bearing if any tool natively honors it.

### Layer 2 — Gates (hooks, deterministic)

The only layer that survives a forgetful or jailbroken agent. Hooks read frontmatter at tool-call time. They don't rely on the agent to remember anything.

For Claude Code, a `PreToolUse` hook on `Read`:

```python
# .claude/hooks/block-restricted-reads.py (sketch)
import json, re, sys
from pathlib import Path

VAULT_ROOTS = ["/path/to/vault"]

def main():
    payload = json.load(sys.stdin)
    file_path = (payload.get("tool_input") or {}).get("file_path", "")
    if not file_path.endswith(".md"): sys.exit(0)
    if not any(file_path.startswith(r) for r in VAULT_ROOTS): sys.exit(0)
    p = Path(file_path)
    if not p.exists(): sys.exit(0)
    head = p.open("r", encoding="utf-8", errors="replace").read(2048)
    if not head.startswith("---\n"): sys.exit(0)
    fm = head[4:head.find("\n---\n", 4)] if "\n---\n" in head else ""
    m = re.search(r"^sensitivity\s*:\s*([^\s#]+)", fm, re.MULTILINE)
    if m and m.group(1).strip("\"'") == "restricted":
        print(json.dumps({"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": (
                f"File '{file_path}' has sensitivity:restricted. Ask the user before reading."
            )
        }}))
    sys.exit(0)
```

Wired into `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [{
          "type": "command",
          "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/block-restricted-reads.py\"",
          "timeout": 5
        }]
      }
    ]
  }
}
```

The hook is cheap (reads at most 2KB), short-circuits early on non-md / non-vault / missing files, and returns a clear `permissionDecision: deny` with a reason. The agent then asks the user. The user authorizes per-task, the agent retries, but **the hook still denies** — it's a hard gate, not a soft prompt. This is intentional: the user's authorization needs to manifest as a different code path (Bash `cat`, see Gaps below), not as the same Read call magically passing the second time.

### Layer 3 — Conventions (CLAUDE.md, honor-system)

Folder-level rules that the agent should honor. Pure honor-system. Useful for things hooks don't cover well: how to think about commits, how to scope skill behavior, what to avoid even when allowed.

Example folder CLAUDE.md ([from the work vault's `personal/role-design/CLAUDE.md`](), private):

```markdown
## Hard rules for any session in this folder

- **Do not** auto-process content here via skills like `/transcript-processor` or `/incorporate`. This is restricted.
- **Do not** include this folder in `Home.md`, Demo-safe Bases views, or anything that could be visible during screen-share.
- **Do not** summarize, reference, or surface this content in any other folder of the vault, in commits, in PRs, or in any output that would be copy-pasted outside this folder.
- **Do** treat anything said related to this initiative with the same care as the artifacts here.
```

These rules are honor-system. They break down with a forgetful agent. Don't put load-bearing enforcement here.

## Observed gaps in real use

Real session, 2026-05-05: a counselor transcript was routed into a `personal/role-design/` folder (sensitivity: restricted). The Read hook fired correctly four times before the user authorized the read explicitly. Gaps observed:

### Gap 1 — The Bash escape hatch

The Read hook gates the `Read` tool. It does not gate `Bash cat`, `Bash python3 ...`, or any MCP tool. The agent who knows the convention will use Read and trip the gate. The agent who doesn't (or one operating in good faith on a user authorization) will reach for Bash and bypass entirely.

**This is a real bypass, not a theoretical one.** When the user explicitly authorized a read in this session, the agent (correctly noting that the Read hook had no override path despite the message suggesting one) used `cat` via Bash. That worked. Same with `python3` for edits.

**UPDATE 2026-07-16 — the platform closed this gap, and the mitigations below are superseded. Do not build the Bash-regex hook.** Claude Code now ships two native controls that did not exist when this was written:

| Layer | What it covers | Enforced by |
|---|---|---|
| `permissions.deny` → `Read(path/**)` | built-in tools **and** file commands Claude Code recognizes in Bash (`cat`, `head`, `tail`, `sed`) | the harness, by rule |
| `sandbox.filesystem.denyRead` | **every** subprocess and child process, whatever the shell syntax | the OS — macOS seatbelt / Linux bubblewrap |

Set the same paths at both layers; they merge. **`sandbox.filesystem.denyRead` is the one that is actually a boundary**, because it never parses the command — a path it cannot see in a string is still a path the kernel refuses to open. Anthropic's own hooks doc now states the rule directly: *"Because the `if` filter is best-effort, use the [permission system](https://code.claude.com/docs/en/permissions) rather than a hook to enforce a hard allow or deny."*

**The third bullet below was the right instinct and only the cost was misjudged.** "Tagging at the filesystem level so readers must opt in to the protected fs" was rejected here as *way too invasive* — that is precisely what `denyRead` does, and it now costs four lines of JSON. Rejecting the correct architecture on a cost estimate that the platform then invalidated is the failure mode worth naming: **re-check the platform before living with a known limitation** (`scan-prior-art-before-building-infra.md`). This gap was documented and tolerated for ~10 weeks after the fix shipped.

~~Mitigations to consider:~~ **Superseded, kept for the reasoning:**

- ~~A second hook on `Bash` matching `cat .*\.md`, `head .*\.md`, `tail .*\.md`, `python.*open\(.*\.md\)`~~ — **empirically insufficient, see the evidence in `router-worker-exfil-containment.md`.** The instinct in the original wording was right ("won't cover everything... the goal is making the bypass *visible*"), and that framing still holds for a hook. It just is not a substitute for the sandbox.
- ~~A wrapper around `cat`/`head`/`tail` that reads frontmatter and denies on restricted.~~ Loses to `python3 -c`, which never invokes the wrapper.
- ~~Tagging at the filesystem level (xattrs or similar)... Way too invasive.~~ **Right answer, wrong cost.** See above.

The pragmatic stance, revised: **the sandbox is the gate; the Read hook is the tier-aware explanation on top of it.** The hook still earns its place — it knows the tier vocabulary, it can say *why* and point at the authorized route, and it fires on structured tools with perfect fidelity. What it can no longer claim is to be the boundary. Keep both: the hook for legibility, the sandbox for enforcement. And keep auditing transcripts — a user-authorized Bash read of a restricted file remains an explicit policy decision, not a quiet workaround.

### Gap 2 — No frontmatter enforcement on new files in restricted folders

When the agent creates a new file in a folder whose convention is "everything here is restricted," nothing forces the new file to carry `sensitivity: restricted` in frontmatter. The convention is visual (other files have it) and conventional (the folder CLAUDE.md says so). A fresh-context agent might miss it.

Mitigations:

- A `PostToolUse` hook on `Write` that, for new files in declared-restricted folders, verifies the frontmatter and **fails the write** if `sensitivity` is missing or set to a lower tier. This is enforcement, not warning.
- A folder-level `.sensitivity` file that hooks read to determine the expected tier for new files in that folder. Avoids hardcoding folder paths into the hook.

### Gap 3 — Commit message content scanning

The folder convention says "don't surface restricted content in commit messages." Nothing checks. The agent commits in good faith but might name a specific person, document, or framing.

Mitigations:

- A `commit-msg` git hook that runs `git diff --cached --name-only` and, for any path in a declared-restricted area, scans the commit message for content references. Hard to scan in general; easier if the convention is "use generic verbs only" — e.g., a regex that allows `update(role-design): ...` but blocks proper nouns, file titles, or recognizable phrases inside the message body.
- A simpler version: any commit touching files with `sensitivity: restricted` triggers a warning that asks the user to confirm the message is generic.

### Gap 4 — Transitive Read→Edit gating

Edit and Write tools require the agent to have used Read on the file first (Claude Code's tool semantics). So the Read hook *implicitly* gates Edit/Write via the harness. But this is a property of one harness, not an enforced rule. A different agent that doesn't enforce read-before-edit would write to restricted files freely.

Mitigations:

- The Read hook should be paired with parallel hooks on Edit and Write that check sensitivity directly. Don't rely on the harness to make Edit safe transitively.

## How to Adopt

1. **Decide the tier vocabulary.** Four values is a working default. Three is fine. More than four becomes hard to remember.
2. **Backfill `sensitivity:` frontmatter** on existing files. A one-time script that walks the vault and asks per-folder. Don't try to guess — set defaults at the folder level (most files in `team/` are `internal`, most in `annual-review/` are `confidential`) and override per-file.
3. **Write the ADR.** Capture the tier semantics and the rationale somewhere durable (e.g., `docs/decisions/YYYY-MM-DD-vault-sensitivity-tiers.md`). The ADR is the source of truth when conventions get questioned.
4. **Ship the Read hook.** Smallest possible script that reads frontmatter and denies on `restricted`. Test it on a known-restricted file and confirm the deny path fires.
5. **Write folder CLAUDE.mds for restricted folders.** Hard rules + a one-line index pointing at the ADR.
6. **Wire into Bases / queries / dashboards.** Default views filter `sensitivity != "confidential"`. Demo-safe views are the default; "All" views are explicit click-through. Nothing in the home page embeds non-demo-safe views.
7. **Audit the gaps.** Run through the four gaps above and decide which to close now (Bash hook? frontmatter-enforcement hook? commit-msg hook? Edit/Write hooks?) versus which to live with as known limitations.
8. **Plan for local-model routing of the restricted tier.** See [`local-model-routing-for-restricted.md`](local-model-routing-for-restricted.md) for the forward-looking pattern.

## Watch-outs

**Sensitivity tagging without hook enforcement is theater.** A vault with `sensitivity: restricted` everywhere but no Read hook is no more protected than one without the property. The property is a prerequisite for enforcement, not a substitute.

**Claude Code does not allow user override of hook denials.** Verified against the [official hooks docs](https://code.claude.com/docs/en/hooks) and [anthropics/claude-code, issue #35136](https://github.com/anthropics/claude-code/issues/35136): there is no slash command, permission mode, or `--dangerously-skip-permissions` flag that bypasses a hook-enforced `deny`. Once the hook denies, that tool call is blocked, period.

The `PermissionDenied` event with `{retry: true}` semantics applies only to *auto-mode classifier* denials, not to hook denials. They look similar but are different paths.

This means a deny message that reads *"retry with a brief justification"* is misleading — there is no retry path through the same tool. The agent must escalate to a different tool. In practice, since the Read hook gates only `Read`, the escape path is Bash (`cat`, `head`, `python3` open). That escape is not a workaround to be hidden; it's the *only* user-authorized path the current single-tool gate affords. Either:
- **Document it honestly in the deny message** (what the work vault hook now does, May 2026), making the Bash bypass an explicit user-authorized path with transcript audit trail, or
- **Build a real out-of-band override** (env var or side file the *user* — not the agent — controls), and gate the parallel tools (Bash `cat`/`head`/`tail` + `python3` reads of `.md`) too. ~~Closes the loophole at the cost of more hook code.~~

**Correction 2026-07-16: "closes the loophole" is false, and this sentence was the most dangerous line in the doc** — it told a reader that more hook code buys a closed boundary. It does not. Gating `python3` reads of `.md` by pattern-matching the command string fails on `python3 -c "print(open('/x/y.md').read())"`, where the path lives inside a program string that no shell parse resolves — verified against a live hook, 2026-07-16. Write more hook code for *legibility*, never for closure; closure comes from `sandbox.filesystem.denyRead` (see Gap 1). The out-of-band override is still worth building — but as an authorization channel, not as a boundary.

The honest-deny-message option is what to ship first. The full out-of-band override is the harder work that requires designing the user-only authorization channel.

**Demo safety is a habit, not a feature.** Even with all of this: the git history contains everything regardless of property; Obsidian's quick-switcher and search find restricted files unless added to "Excluded Files"; open tabs persist sensitive content. The single highest-value demo habit is `Cmd-W` everything before sharing a screen, then opening the home page fresh.

**Restricted ≠ secret.** The `restricted` tier signals *don't process this casually*, not *cryptographically inaccessible*. For truly secret content (credentials, financial data) you want a different mechanism — separate vault, age/sops encryption, hardware keys. The sensitivity-tier system protects against the dominant failure modes (forgetful agent, accidental demo leak, casual cloud upload), not against an adversary with read access to the filesystem.

## Adjacent Patterns

- **[`local-model-routing-for-restricted.md`](local-model-routing-for-restricted.md)** — the forward-looking direction: when local models are capable enough, route restricted-tier reads/processing to a local model that never crosses the network boundary. The sensitivity property becomes the routing key.
- **[`pre-push-pr-discipline-hook.md`](pre-push-pr-discipline-hook.md)** — same shape (mechanical enforcement at the moment of action), different domain (PR discipline). Both are examples of "encode the rule as code, not as memory."
- **[`wire-into-existing-flows.md`](wire-into-existing-flows.md)** — the sensitivity ADR alone is a standalone artifact. The Read hook is the wiring. Don't ship the ADR without the hook.
- **[`decision-doc-adr.md`](decision-doc-adr.md)** — sensitivity tiers are a non-trivial choice that benefit from an ADR. Future-you needs to know *why* there are four values, not three or five.

## Also seen: 2026-09-11, the speed bump starves the trusted lane

The personal agent's desk grant opened a private vault to direct `Read` and `Write`, and kept the Bash hook's rule that any shell command naming the vault folder counts as a bulk read, since it can walk every note, restricted ones included, where the frontmatter check cannot see. The rule is right about the risk and blind to the verb. `git -C <vault> status`, `cd <vault> && git commit -- one-note.md` and `ls <dir> | tail` were all denied exactly like `cat <dir>/*`, because the hook matches the path and allows only a fixed set of listing verbs, and only with no pipe. That made it half a grant: a note could be written at the desk but never committed, and nothing reported the gap until a live session tried it. When a wholesale Bash deny sits under a grant, list the operations the trusted lane actually needs (commit is the obvious one) and allow them by the verb's own grammar. Git's names-only subcommands pass, and anything that prints a note body stays denied. Test a grant by doing the whole job once, from the write through the commit, not just the first step.
