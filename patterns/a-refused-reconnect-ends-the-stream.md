---
type: pattern
date: "2026-08-13"
source: The dashboard live dogfooding — a server restart 6s long left the open page's EventSource permanently CLOSED; the page kept narrating the old brain's state with total confidence
tags:
  - code
  - reliability
  - event-driven
  - frontend
  - anti-pattern
---

# A Refused Reconnect Ends the Stream

Transport-level auto-reconnect certifies only the outages shorter than its
retry policy. Browsers reconnect an EventSource that *drops* — but when a
retry lands while the server is still down, the connection is **refused**,
and Chromium marks the stream CLOSED permanently. No further retries, no
error UI, nothing. Any server restart slower than one retry window (a few
seconds) leaves every open page deaf forever.

The failure is invisible by construction: an event-driven page that stops
receiving events doesn't look broken, it looks like *nothing has changed*.
The dashboard kept showing a stale unsaved-files list — confidently, in the
designed styling — while the real state had moved twice. And it never
appears in testing, because test restarts are fast: kill-and-relaunch in one
command fits inside the retry window, so the built-in reconnect works every
time you watch it.

## The Pattern

Own recovery **above** the transport, and make the recovery decision a
liveness read, not a retry loop:

```js
es.onerror = () => {
  if (es.readyState !== EventSource.CLOSED) return; // browser still retrying
  es.close();
  checkHealth().then(connectEvents)     // alive → a fresh stream
    .catch(() => setAwakeUI(false));    // dead → the designed downstate
};
```

- **One health read decides.** Server alive → open a new stream. Server dead
  → show the honest "asleep" state with the wake instructions. Never a
  quiet `setTimeout` retry loop — that is polling the port, and it keeps the
  page *looking* healthy while deaf.
- **Recovery is reconnect + reconcile, not just reconnect.** The first
  message on every stream carries the server's identity (pid) and standing
  state; a recovered stream that finds a *different* server resyncs or
  reloads instead of splicing new events onto a stale picture. (Sibling of
  `state-published-as-an-event-is-lost-to-latecomers`: the connect moment
  must re-send truth, because the latecomer here is your own page, five
  seconds older.)
- **The user-initiated revive path must re-arm the stream.** The "check
  again" button that wakes the UI has to reopen the event stream too, or it
  restores the picture and leaves the deafness.
- Close the old object before opening a new one — two live streams narrating
  one page is its own bug.

The general shape outlives SSE: any client with built-in retry (WebSocket
libraries, message-bus consumers, file watchers on remounted volumes) has a
policy edge past which it silently gives up. Find the edge by outage length,
not by reading docs — restart the producer *slower* than you naturally
would, then check whether the consumer ever hears again.
