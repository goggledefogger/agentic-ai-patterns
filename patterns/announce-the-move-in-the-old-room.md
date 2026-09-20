---
type: pattern
date: "2026-08-01"
source: The personal agent — walk-and-talk tmux twin moved the conversation while the user's remote-control phone UI stayed bound to the original session, frozen at "Phone's on the tailnet…"
tags:
  - multi-surface
  - handoff
  - agents
  - ui
  - anti-pattern
---

# Announce the Move in the Old Room

An assistant can relocate its own conversation: fork it into a tmux twin so a
phone can inject input, hand off to a fresh session, resume itself elsewhere.
The work continues seamlessly — for the assistant. Every UI bound to the old
location has no idea. It does not error, does not redirect, does not say
"continued elsewhere." It shows the last pre-move message, forever, while the
link the user needs lands in a room they are not in.

## The Problem

A walk-and-talk session was started from the Claude Code **remote-control phone
UI**. The skill's (correct, for terminals) directive told the assistant to twin
its conversation into tmux so the audio bridge could inject voice input. The
twin worked. But the user's phone UI is bound to the *original* session — so his
app froze at *"Phone's on the tailnet…"*, the last message before the move,
while the twin printed the page URL, ran the briefing, and waited for him.

The user's report was exactly the symptom class this produces: **"those messages
aren't making it there."** Nothing was broken; the conversation had moved and
nothing said so *in the place he was looking*.

Two aggravators, same family:

- **Tool output is not a delivery surface.** The URL had printed twice — inside
  Bash results, which the remote-control UI collapses. Visible in a terminal,
  nonexistent on the phone. (A QR code is worse: it is a Mac-screen-only
  artifact by construction.)
- **Two live brains.** After the twin, the page mic fed the twin while app
  typing fed the original — two assistants on one vault, each seeing half the
  user's inputs.

## The Pattern

**Before relocating a conversation, send one last message in the old room:**

1. **Say the view is about to freeze** — "this chat goes quiet now; the walk
   continues on the page."
2. **Carry the pointer in reply text** — the bare URL as tappable text in the
   message body. Not in tool output, not as a QR, not "see above."
3. **Name the one brain** — which surface the user should talk to from now on,
   and make the abandoned one inert (or say plainly it is history).

The general rule under it: **a must-reach artifact lives in the message body of
the surface the user is actually looking at, or it was never delivered.** The
sender's context shows both surfaces; the user's shows one. Delivery is judged
from the user's side.

## Corollary for Injection

Driving another session's input programmatically is a solved-once problem per
TUI — Claude Code's paste detection strands a `tmux send-keys "<text>" Enter`
as unsubmitted input that looks exactly like being ignored. If a project ships
an inject script, free-handing the primitive re-finds the bug it exists for.

## The Rule Did Not Hold on Its Own (2026-08-02)

This entry was written on 2026-08-01. The next morning the same assistant, on
the same skill, did the same thing: brought the tunnel up, twinned the session,
launched the bridge, ended the turn — and the walker on the phone app saw
nothing and had to ask for the link. The URL had printed twice, both times in
tool output. **The rule was already written, in two places, and lost anyway.**

That is the real finding, and it generalizes past this incident: *a
must-reach-the-user obligation is a turn-boundary property, and prose cannot
enforce a turn boundary.* The obligation is invisible at the moment it is
violated — nothing errors, every step succeeded, and the assistant's own
context shows the URL twice. Only the ending of the turn is the observable
event.

So it moved to a control, the same move the two-way loop check made after being
walked past in the same way: **a Stop hook that refuses to end the turn while a
live session's link has never appeared in an assistant text block.** Two design
points carried the weight, and both are reusable:

- **Parse the transcript, don't grep it.** The URL's token appears throughout a
  setup transcript inside tool inputs and results. A substring search over the
  file passes on the assistant's own plumbing — a [[decorative-gate]], whose
  success branch is reachable without the thing being true. Only an assistant
  *text* block is delivery. (Substring is still sound for proving *absence*, so
  it makes a fine fast path.)
- **Fail open, and know why.** Most gates here fail closed. This one blocks the
  assistant rather than a dangerous action, and the harm it prevents is
  silence — so an unreadable transcript that wedged the session would be a
  worse instance of the failure itself. The direction of the damage picks the
  default, not habit.

## Related

- [[external-file-visibility]] — artifacts outside the surface the reader opens
- [[eyes-free-approval-channel]] — surfacing a blocking prompt where the human is
- [[log-the-verdict-not-the-volume]] — the sibling rule for machine-facing surfaces

## The Rule of Thumb

If you moved the conversation and the user's next message arrives in the old
room, they never saw the move. Design so that cannot happen silently: the move
announcement is the last message the old room ever shows.
