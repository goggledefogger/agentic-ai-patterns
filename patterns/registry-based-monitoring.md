---
type: pattern
date: "2026-03-27"
source: A course participant's home-agent system (the scheduled-tasks repo)
tags:
  - monitoring
  - automation
  - scripts
---

# Registry-Based System Monitoring

A JSON registry of all agents, scheduled tasks, and background jobs, with a script that checks each one against expected behavior and auto-discovers unregistered items.

## The Pattern

Instead of hardcoding health checks for each job, maintain a `registry.json` that describes every agent:

```json
{
  "launchd_agents": [
    {
      "name": "com.example.interactions",
      "description": "Gmail, iMessage, WhatsApp → Airtable CRM",
      "log_path": "~/logs/interactions.log",
      "max_log_age_minutes": 30,
      "error_patterns": ["ERROR", "AuthError", "FAILED"]
    }
  ],
  "writers": [
    {
      "name": "interaction-capture",
      "description": "Single writer for Interactions table",
      "expected_interval_hours": 0.5,
      "log_path": "~/logs/interactions.log"
    }
  ]
}
```

A single Python script reads the registry, checks each item (is it running? is the log fresh? are there errors?), and reports status with color-coded output.

## Why It Works

- **Adding a new agent** = adding a JSON entry. No script changes needed
- **Auto-discovery** catches jobs someone added but forgot to register — the script scans crontab, LaunchAgents, and task directories for anything not in the registry and flags it as `[NEW]`
- **Writer health** checks file modification times against expected intervals. If a single-writer agent hasn't run in 2x its expected interval, it alerts. Catches silent failures from Mac sleep, token expiry, or OS updates
- **Multiple output modes**: full color report, JSON (for piping to Slack), or one-line-per-job brief

## When to Use

Any project with 3+ scheduled tasks or background processes. The registry pays for itself the first time it catches a job that silently stopped running.

## Source

Derived from `system-health-check/health_check.py` (280 lines, Python). The original monitors cron jobs, launchd agents, Claude Code scheduled tasks, single-writer agents, GitHub repos, and system resources (disk space, uptime).
