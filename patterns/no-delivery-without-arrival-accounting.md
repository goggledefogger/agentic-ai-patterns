---
type: pattern
date: "2026-08-03"
source: The personal agent's walk-and-talk skill (ux-debt 61-63, 69, 70 + 2026-08-23 — one bug shape, seven channels)
tags:
  - code
  - reliability
  - messaging
  - voice
---

# No Delivery Without Arrival Accounting

Every channel that carries something to a human (or into another process) must answer two questions **at design time**: *how do I learn it arrived?* and *how many times may it fire?* A send with no arrival signal will eventually fail silently; a retry or fallback with no arrival signal will eventually fire twice. Both were found live, repeatedly, in one week of a voice assistant's field use — the same shape in five different channels.

## The eight instances

1. **The link push.** The URL a walker needs was written to disk and spoken into a conversation — and the conversation is the least reliable pipe in the system. Twice in one day the link existed everywhere except the walker's phone. Fix: the machine that mints the URL delivers it (Telegram text push), and the file is rewritten-or-cleared so a stale copy cannot impersonate delivery.
2. **The speech that fired twice.** Text-to-speech went to the web page *and* a Telegram voice note in parallel — five duplicate notes in one walk. The page reports playback per utterance, so the question "did they hear it?" was answerable; the second channel just never asked. Fix: fallback, never parallel — the note fires only if no playback is reported within a grace window.
3. **The turn that never ended.** One assistant turn pre-produced handshake, brief, and answers; the walker's replies arrived mid-turn and every queued utterance played whenever it landed — back-to-back monologue with no proof the walker's words registered. Fix: speak once, end the turn; nothing that depends on a reply may be pre-produced.
4. **The gates that could never pass.** Three always-false checks (a JSON schema path the vendor had changed, a mangled probe URL, a grep for a word the endpoint never returns) sat *in front of* delivery and each degraded politely — no push, no write, a stale file passing every downstream check. Fix: deliver on the deterministic fact (the serve config), probe *after* the send, and let the probe only decide wording. **A gate in front of delivery must fail loud or not exist.**
5. **Input injection into a wedged UI.** Voice input was typed into a TUI via `tmux send-keys` — keystrokes rendered in the input box while the frozen process consumed nothing, for seven minutes. Dispatch was confirmed; consumption never was. Fix: injection verifies arrival (the transcript grew), not keystroke delivery, and a watchdog treats "input injected, transcript static" as a dead session.
6. **The log entry written before the act** (2026-08-23). The bridge stamped `kind: injected` into its input log *before* running the injection — so words spoken while no session existed at all were logged as delivered, and the silence had no witness. A log entry written before the act is a **prediction wearing a record's clothes**; when the act fails, the log swears it happened. Fix: the entry follows the verdict — success logs `injected`, failure logs `inject-failed` *and tells the human out loud*.
7. **The arrival probe that reads the wrong region** (2026-08-23, same hour). The injection-arrival check from instance 5 grepped only the last prompt line of the TUI for the injected text — but long text *wraps* onto lines below the prompt, and paste-collapse replaces it with a `[Pasted text #N]` placeholder. Both false-passed: probe says consumed, text sits stranded. An arrival check is itself a channel claim, and it can lie the same way the send can — verify the probe observes the *whole* surface the failure can occupy, not the corner where it was first seen.
8. **The evidence that matched by prefix** (2026-09-11). A delivery lane closed only when the send log held a line proving the send — and it looked for that line by searching the log for the task's URL as a substring. The first receipt ever sent to the *list* page closed as "delivered" without sending: the list URL is a prefix of every report link ever sent, so a report delivered nine hours earlier stood as evidence. The arrival check was sound in shape and wrong in key. Fix: evidence is matched by the exact message this task would send (or the envelope key written into the log line), never by a URL or a slug that another delivery can contain. The lane had been in production for weeks and had never once been exercised on the case that broke it; the "proof" the first live test produced was the bug.

## The Pattern

For each channel, write down:

- **Arrival signal** — the observable fact that proves the message landed (playback beacon, transcript growth, HTTP 200 from the *receiver's* side, the human's reply). If none exists, build one or demote the channel.
- **Fire policy** — exactly once, or fallback-after-silence. Parallel delivery to one recipient is almost always a bug: the duplicate is pure noise and the duplication *hides* which channel actually works.
- **Staleness rule** — any cached copy of the deliverable (a URL file, a replay buffer) is refreshed or cleared by the process that owns it, so it cannot outlive its truth and satisfy a checker.

The test for whether you have this right: when delivery fails, does anything *say so*? Silent non-delivery is the worst outcome on every channel — worse than duplicates, worse than noise — because the sender's evidence all says it worked.

## When it applies

Notifications, chat bridges, TTS/voice surfaces, webhooks, queue consumers, IPC into TUIs or GUIs — anywhere the happy path is "we sent it" and nobody owns "they got it."
