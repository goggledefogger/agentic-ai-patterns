---
type: pattern
date: "2026-10-09"
source: A local dashboard's "asleep" screen gained a recovery watcher; code review found the first draft could reload the page about once a second, forever
tags:
  - code
  - debugging
  - anti-pattern
  - reliability
---

# A Guard for a Loop That Crosses Reloads Lives Outside Memory

A dashboard shows an "asleep" screen when its server is down. It got a recovery watcher: probe the health endpoint, and when the server answers, reload the page. Four paths could fire it at once: the timer, a button, window focus, and tab visibility.

The first draft had one try/catch. It wrapped the health read and also about 100 lines of rendering that followed. So a rendering error, anywhere in those lines, landed in the same catch as "server unreachable". The page reloaded, ran the same code, threw the same error, and reloaded again. About once a second, with no limit.

## Why the wrong move was attractive

The catch looked like tidy error handling: if anything goes wrong, try again. The obvious guard, a counter, felt like one line. But a counter held in a variable is wiped by the very reload it is supposed to count. Every new page load starts at zero, so the budget is never spent.

## The rule

**A loop that spans page loads (or process restarts) needs its budget in storage that survives the reload, and its trigger must separate "the service is back" from "this page works".**

What the fix did:

1. **The catch holds only the health read.** Rendering errors are no longer swallowed into the retry path.
2. **A reload budget of 3 per 60 seconds, kept in `sessionStorage`.** It survives the reload, so the third attempt is the last.
3. **A single "reloading" flag**, so the four trigger paths cannot each start their own reload.
4. **A "boot broken" flag**, set when rendering fails. A page that cannot render never auto-reloads; it waits for a person.

## The deeper tell

A server answering its health endpoint is not the page being able to do its job. The watcher treated "the server answers" as "reloading will help". When the page itself is broken, the answer is yes, the server is fine, and reloading cannot help. This is the same gap as [a health check that never exercises the work](health-check-that-never-exercises.md), seen from the consumer side: the signal says the dependency is up, and nothing says the thing you are about to do will succeed.

**The tell:** a retry whose only memory is a local variable, in a process that the retry itself replaces. Ask what the counter is stored in, and what the retry does to it.

## Adjacent Patterns

- [A Health Check That Never Exercises the Work](health-check-that-never-exercises.md): the signal this watcher trusted.
- [A Guard's Evidence Must Outlive the Failure It Detects](guard-evidence-outlives-the-failure.md): the same storage question, one level over.
