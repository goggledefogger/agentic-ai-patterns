---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo, commit 169ae59)
tags:
  - security
  - tunneling
  - auth
---

# Static-Token Tunnel Auth Gate

When exposing a local dev server through a tunnel (Cloudflare, ngrok, Tailscale funnel), gate requests behind a static token header, and detect tunnel-vs-local via a tunnel-injected header so local dev bypasses the check.

## The Problem

Exposing a local dev server through a public tunnel to share a preview, demo to a client, or hit from a phone creates an open endpoint. Anyone with the URL can hit it. "The URL is hard to guess" is not a security model.

Proper auth (Clerk, Descope, Auth0) is the right long-term answer, but it's overkill for solo dev work or short-lived previews. The gap between "raw exposure" and "proper auth" wants a minimal viable auth layer: ~80 lines of middleware, a static token, and localhost bypass.

## The Pattern

1. Generate a random token, put it in an env var.
2. Add a middleware (`onRequest` hook in Fastify, `before_request` in Flask, middleware in Express) that:
   - Reads a header like `x-app-token` from the request.
   - If the request came from the tunnel (detected via a tunnel-injected header like `Cf-Connecting-Ip` for Cloudflare, `X-Forwarded-For` for ngrok), require the token match.
   - If the request is local (no tunnel-injected header), pass through.
3. Configure the client (dev proxy, browser extension, or CLI) to attach the header.

Localhost bypass means `npm run dev` still works without passing headers. Only tunnel traffic is gated.

## Example

From the participant's cockpit at `cockpit/proxy/`:

```js
// Fastify onRequest hook
fastify.addHook("onRequest", async (req, reply) => {
  // Local dev proxy hits 127.0.0.1 directly — no tunnel headers.
  const viaTunnel = req.headers["cf-connecting-ip"];
  if (!viaTunnel) return;

  if (req.url.startsWith("/api/")) {
    const token = req.headers["x-cockpit-token"];
    if (token !== process.env.COCKPIT_TOKEN) {
      reply.code(401).send({ error: "unauthorized" });
    }
  }
});
```

The browser-side client at `cockpit/src/lib/proxy.ts` adds the header automatically when the request is going through the tunnel URL.

## Flask Equivalent

```python
from flask import request, abort

@app.before_request
def check_tunnel_auth():
    via_tunnel = request.headers.get("Cf-Connecting-Ip")
    if not via_tunnel:
        return  # local, pass

    if request.path.startswith("/api/"):
        if request.headers.get("X-App-Token") != os.environ["APP_TOKEN"]:
            abort(401)
```

## When to Use

- Sharing a local dev server with a collaborator, client, or phone via tunnel.
- Demo deployments that shouldn't be public but don't warrant a full auth system.
- Internal-only services behind a tunnel (admin dashboards, ops tools).

## When NOT to Use

- Production services with real users. Use Clerk / Descope / Auth0 / Supabase Auth.
- Multi-user systems where you need to know *which* user is calling.
- Anything handling personal data, payments, or irreversible actions.

## Upgrade Path

When the static-token gate is no longer enough (multi-user, audit trail, MFA, etc.), the natural upgrade is:

- For Next.js / React on Vercel: Clerk via the Vercel Marketplace (auto-provisioned env vars, middleware-based gating).
- For Flask / custom: Auth0 or Descope with OIDC.

The static-token gate is a **stepping stone**, not a destination. The pattern exists so you can ship the preview today and graduate to proper auth when you actually need it.

## Adjacent Patterns

- Pairs with decision-doc pattern — write a short decision doc capturing "we chose static token because X" so the next maintainer knows when to upgrade.
