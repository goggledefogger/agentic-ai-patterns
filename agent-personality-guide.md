---
type: reference
date: "2026-03-26"
tags:
  - meta
  - claude-code
  - agent
  - personality
---

# Agent Personality: SOUL.md + Claude Code

How to extend an Obsidian + Claude Code vault into a persistent agent with its own identity, voice, and GitHub presence. Inspired by the OpenClaw framework's approach to agent personality files.

## The Idea

A vault is already a brain, it holds context, tracks projects, runs commands. Adding a SOUL.md file gives it a personality. Adding a dedicated GitHub account gives it a presence. Adding hooks and scripts gives it guardrails so the personality stays consistent and the identity never leaks.

This isn't about making Claude "act like a character." It's about creating a persistent agent identity that:
- Survives across sessions (SOUL.md is read every time)
- Has opinions and a voice (not generic AI output)
- Interacts on GitHub under its own account (not yours)
- Stays in its lane (hooks and scripts enforce boundaries)

## The Files

### SOUL.md — Who the agent is

The most important file. Every session starts by reading it. It defines:

**Identity and worldview:**
- What the agent cares about, what it ignores
- Strong opinions (not hedging, commit to takes)
- Values and priorities
- What it optimizes for

**Voice:**
- How it talks (specific, not "be friendly")
- Example output showing good and bad
- Anti-patterns to avoid

**Boundaries:**
- What it does and doesn't do
- Who it serves
- What requires human approval

A good SOUL.md is under 200 lines. Someone reading it should be able to predict the agent's take on a new topic. If they can't, it's too vague.

### STYLE.md — How the agent communicates

Separate from SOUL.md because style is more granular:
- Formatting rules (headers, code blocks, emoji policy)
- Length constraints
- Medium-specific voice (GitHub review vs Slack vs email)
- Example output with commentary on why it works

### CLAUDE.md — How the agent operates

The operational layer. References SOUL.md and STYLE.md, then covers:
- Session start protocol (read soul → check projects → verify identity)
- Tool access and constraints
- Identity separation rules
- Project-specific instructions

### Priority hierarchy with personality

The vault's priority hierarchy (scripts > obsidian markdown > skills > docs > memory) still governs where work products live. For the personality files specifically, precedence runs:

1. **Scripts/code**, deterministic identity enforcement, hooks, wrapper scripts
2. **Obsidian markdown**, vault notes, project context, activity logs
3. **SOUL.md / STYLE.md**, personality and voice (read every session, not edited by the agent)
4. **CLAUDE.md**, operational instructions
5. **Memory**, last resort, for cross-session context that doesn't fit elsewhere

SOUL.md sits above CLAUDE.md because personality should inform operations, not the other way around.

## Identity Separation

The hardest part of running an agent with its own GitHub account is making sure identities never cross. Three layers of enforcement:

### Layer 1: Local git config (repo-level)

Set the agent's identity in every repo it touches:

```bash
git -C /path/to/repo config user.name "agent-name"
git -C /path/to/repo config user.email "agent-name@users.noreply.github.com"
```

This means even accidental commits use the right identity. Your global git config (your personal identity) stays untouched.

### Layer 2: Pre-push hooks

Install a hook in every repo the agent pushes to:

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push
EXPECTED_USER="agent-name"
ACTUAL_USER=$(git config user.name)

if [ "$ACTUAL_USER" != "$EXPECTED_USER" ]; then
  echo "ERROR: Refusing to push as '$ACTUAL_USER' — expected '$EXPECTED_USER'"
  exit 1
fi
```

This catches the case where git config got reset (OS update, repo re-clone, etc).

### Layer 3: Wrapper script

A single script that handles all git and gh operations for the agent:

```bash
#!/usr/bin/env bash
# scripts/agent.sh

AGENT_USER="agent-name"
AGENT_EMAIL="agent-name@users.noreply.github.com"

get_token() {
  security find-generic-password -a "$AGENT_USER" -s "github-pat" -w 2>/dev/null || {
    echo "ERROR: token not found in Keychain" >&2; exit 1
  }
}

# gh CLI — always uses agent's token
cmd_gh() {
  GH_TOKEN=$(get_token) gh "$@"
}

# git commit — always uses agent's identity
cmd_commit() {
  local repo="$1" msg="$2"
  git -C "$repo" -c user.name="$AGENT_USER" -c user.email="$AGENT_EMAIL" commit -m "$msg"
}

# git push — uses agent's token for auth
cmd_push() {
  local repo="$1" branch="$2" token
  token=$(get_token)
  git -C "$repo" \
    -c user.name="$AGENT_USER" -c user.email="$AGENT_EMAIL" \
    -c "url.https://${AGENT_USER}:${token}@github.com/.insteadOf=https://github.com/" \
    push origin "$branch"
}

# verify — compare the agent's token identity against your default gh identity
cmd_verify() {
  echo "=== agent ==="
  GH_TOKEN=$(get_token) gh api /user --jq '.login'
  echo "=== your account (default) ==="
  gh api /user --jq '.login'
}

# dispatcher — without this, calling the script does nothing
cmd="${1:?Usage: agent.sh {gh|commit|push|verify}}"
shift
case "$cmd" in
  gh)     cmd_gh "$@" ;;
  commit) cmd_commit "$@" ;;
  push)   cmd_push "$@" ;;
  verify) cmd_verify ;;
  *) echo "Unknown command: $cmd (expected gh|commit|push|verify)" >&2; exit 1 ;;
esac
```

One caveat on `cmd_push`: the token lands in the git process's argument list, so it's visible in `ps` output while the push runs. Fine on a single-user machine. On a shared machine, use a git credential helper instead of the `insteadOf` rewrite.

The wrapper means Claude Code commands never construct raw `gh` or `git push` calls, they always go through the script, which always enforces the right identity.

### Token storage

Use macOS Keychain (or your OS equivalent), not environment variables or files:

```bash
# Store
security add-generic-password -a "agent-name" -s "github-pat" -w "<PAT_VALUE>"

# Retrieve
security find-generic-password -a "agent-name" -s "github-pat" -w
```

This keeps the token out of your shell profile, dotfiles, and git history. The wrapper script retrieves it at runtime.

## CLAUDE.md Integration

Your vault's CLAUDE.md should reference the agent identity:

````markdown
## Agent Identity
This vault operates as `agent-name` on GitHub. All git/gh operations use the wrapper:
```bash
AGENT="./scripts/agent.sh"
"$AGENT" gh <args>            # gh CLI as agent
"$AGENT" commit <repo> "msg"  # commit as agent
"$AGENT" push <repo> branch   # push as agent
"$AGENT" verify               # check identities
```
Never use raw `gh` or `git push` for repos this agent manages.
````

## Session Start Protocol

Add to CLAUDE.md:

```markdown
## Session Start
1. Read SOUL.md — remember who you are
2. Read STYLE.md — remember how you talk
3. Run `./scripts/agent.sh verify` — confirm identity separation
4. Check projects/ for active context
5. Proceed with the task
```

The verify step is non-negotiable. Identity drift is the most common failure mode, especially after OS updates, token rotations, or repo re-clones.

## Repo Structure

An agent personality repo looks like:

```
agent-name/
  SOUL.md           — identity, worldview, opinions, values
  STYLE.md          — voice, tone, formatting rules
  CLAUDE.md         — operational instructions
  README.md         — what this is (for humans browsing GitHub)
  projects/         — per-project context
    project-a/
      context.md    — overview, people, repos
  scripts/          — wrapper scripts, hooks
  logs/             — activity logs (optional)
```

Keep it as a private GitHub repo under the agent's own account. This is the agent's home, its persistent context store.

## Writing a Good SOUL.md

### Do

- **Be specific:** "Use dry, understated humor. Mitch Hedberg, not Jim Carrey" not "be funny"
- **Commit to opinions:** "Tests that don't exist are worse than bad tests" not "testing is important"
- **Include anti-patterns:** What the agent should never do is as important as what it should
- **Show, don't tell:** Include example output showing good and bad
- **Keep it under 200 lines:** If it's longer, the personality is too complex to be consistent
- **Include contradictions:** Real personality isn't perfectly consistent. A pirate who cares about shipping velocity but also respects careful architecture is more authentic than one who's purely "move fast"

### Don't

- **Don't be vague:** "Be helpful and friendly" describes every customer service bot. Be distinctive
- **Don't over-constrain:** Leave room for the agent to exercise judgment within its personality
- **Don't make it a system prompt dump:** SOUL.md is identity, not instructions. Operational details go in CLAUDE.md
- **Don't forget to test:** Have the agent review something and check if the output matches the personality. Adjust if it doesn't

## Multiple Personas in One Agent

An agent can have multiple voices (e.g., a product reviewer and a code reviewer). The SOUL.md defines both and explains when each applies. Key rules:

- Each persona has a clear domain, they don't overlap
- Both appear in every output, in consistent order
- Either can opt out (silence) if they have nothing to say
- Persona voice stays out of structured fields (issue titles, branch names), those stay plain English for searchability

## Future: Multi-Channel Agents

This guide covers GitHub as the primary channel. Future extensions:
- Slack (via MCP or webhook)
- Scheduled tasks (Claude Code scheduled tasks, launchd)
- Email
- Obsidian vault as direct interaction surface

The SOUL.md and identity separation patterns apply regardless of channel. The wrapper script extends to cover new auth mechanisms.
