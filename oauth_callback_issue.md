# OAuth Callback Mismatch — Why Claude's Auth Lands on Cursor

## Overview

When authenticating from **Claude** against the hosted MCP server, the OAuth
callback is delivered to **Cursor** instead of Claude. This document explains
exactly why this happens, traces the full flow with concrete examples, and
describes the fix.

---

## Background — How the Auth Stack Works

The MCP server uses a three-layer auth architecture:

```
MCP Client (Cursor / Claude)
        │
        │  HTTPS tool calls
        ▼
CF Worker OAuth Shim          ← handles /authorize, /token, /register
        │
        │  forwards CF JWT as Bearer token
        ▼
MCP Server (your K8s pod)     ← CFAuthMiddleware validates CF JWT
        │
        │  Laravel Passport token exchange (per user)
        ▼
Laravel Dashboard API
```

The **CF Worker** is the OAuth Authorization Server. Its URL is published via
the MCP server's discovery endpoint:

```
GET /.well-known/oauth-authorization-server
→ {
    "authorization_endpoint": "https://leadgen-mcp-oauth.workers.dev/authorize",
    "token_endpoint":         "https://leadgen-mcp-oauth.workers.dev/token",
    "registration_endpoint":  "https://leadgen-mcp-oauth.workers.dev/register",
    ...
  }
```

Both Cursor and Claude discover this same document and use the same Worker
endpoints.

---

## The OAuth Authorization Code Flow (Normal Case)

Here is what happens when a client authenticates correctly:

```
Step 1: Client discovers OAuth endpoints
  → GET https://mcp.example.com/.well-known/oauth-authorization-server

Step 2: Client builds authorization URL with ITS OWN redirect_uri
  → https://leadgen-mcp-oauth.workers.dev/authorize
      ?client_id=<client_id>
      &redirect_uri=https://cursor.example.com/oauth/callback   ← Cursor's own URL
      &response_type=code
      &code_challenge=<PKCE>
      &state=<random>

Step 3: User logs in on CF Access

Step 4: CF Worker sends the auth code back to redirect_uri
  → https://cursor.example.com/oauth/callback?code=abc123&state=<random>

Step 5: Client exchanges code for token
  → POST https://leadgen-mcp-oauth.workers.dev/token
      { code, redirect_uri, client_id, code_verifier }

Step 6: Worker returns a CF JWT (access_token)
  → { access_token: "<CF JWT>", ... }

Step 7: Client uses CF JWT as Bearer token for all tool calls
  → Authorization: Bearer <CF JWT>
```

This works cleanly as long as each client registers its **own** `redirect_uri`
and the Worker validates that the `redirect_uri` at token exchange matches the
one used at authorization.

---

## Where It Goes Wrong — The Callback Lands on Cursor Instead of Claude

### Root Cause: Single Hardcoded `redirect_uri` in the CF Worker

The CF Worker (`/authorize` and `/token` endpoints) is configured with a
**single, hardcoded** `redirect_uri` whitelist — typically set to Cursor's
callback URL because Cursor was the first client integrated.

```
Worker config (pseudocode):
  ALLOWED_REDIRECT_URIS = ["https://cursor.example.com/oauth/callback"]
```

When Claude tries to authenticate, it sends its own `redirect_uri`:

```
Claude → GET /authorize
           ?client_id=...
           &redirect_uri=https://claude.ai/oauth/callback   ← Claude's URL
           &...
```

The Worker sees `redirect_uri=https://claude.ai/oauth/callback` but only
knows about `https://cursor.example.com/oauth/callback`. Two things can happen:

**Scenario A — Worker ignores the requested redirect_uri and uses its hardcoded one:**
```
User authenticates on CF Access
       ↓
Worker fires callback to: https://cursor.example.com/oauth/callback?code=...
       ↓
Cursor receives the auth code meant for Claude
       ↓
Claude is still waiting — its callback URL never gets called
```

**Scenario B — Dynamic Client Registration is not properly scoped:**
The `/register` endpoint creates a new client but stores Cursor's
`redirect_uri` as the default (or the Worker re-uses the same CF Access
application AUD tag for all clients, meaning the token issued is valid for
Cursor's audience, not Claude's).

---

## Concrete Example — Full Broken Flow

**Setup:**
- MCP server: `https://mcp.example.com`
- CF Worker: `https://leadgen-mcp-oauth.workers.dev`
- Cursor callback: `https://cursor.example.com/oauth/callback`
- Claude callback: `https://claude.ai/oauth/callback`
- Worker hardcoded redirect: `https://cursor.example.com/oauth/callback`

**What you do:** Open Claude, add `https://mcp.example.com` as an MCP server,
click Connect.

**Step-by-step breakdown:**

```
1. Claude fetches discovery doc
   GET https://mcp.example.com/.well-known/oauth-authorization-server
   ← { authorization_endpoint: "https://leadgen-mcp-oauth.workers.dev/authorize", ... }

2. Claude registers itself (dynamic client registration)
   POST https://leadgen-mcp-oauth.workers.dev/register
   { redirect_uris: ["https://claude.ai/oauth/callback"] }
   ← { client_id: "claude-xyz", ... }

3. Claude opens the login URL in your browser
   https://leadgen-mcp-oauth.workers.dev/authorize
     ?client_id=claude-xyz
     &redirect_uri=https://claude.ai/oauth/callback    ← Claude wants its own callback
     &response_type=code
     &code_challenge=ABC123
     &state=random-state-456

4. You log in via CF Access (browser)
   → CF validates your identity, issues an auth code

5. ⚠️  Worker fires the callback to the HARDCODED redirect_uri (Cursor's):
   GET https://cursor.example.com/oauth/callback
     ?code=xyz789
     &state=random-state-456

   ← Cursor receives this! Cursor did not initiate this flow.
   ← Claude's callback (https://claude.ai/oauth/callback) is never called.
   ← Claude times out waiting for the auth code.

6. Cursor may try to exchange the code (if it's listening), gets confused
   because the state=random-state-456 doesn't match any pending Cursor flow.

Result: Neither client gets a working token. Or Cursor gets a token it
didn't ask for. Claude gets nothing.
```

---

## Why the Browser Session Makes It Worse

If you authenticated in Cursor first, your browser has a **Cloudflare Access
session cookie** for `mcp.example.com`.

When Claude opens the authorization URL, CF Access sees the existing cookie,
**skips the login page entirely**, and immediately fires the callback — straight
to Cursor's hardcoded URL. The redirect happens so fast you barely notice any
login step occurred, making the bug even harder to spot.

```
Claude opens /authorize in browser
       ↓
CF Access: "I already know this user" (cookie from Cursor session)
       ↓
Immediately redirects to: https://cursor.example.com/oauth/callback?code=...
       ↓
Cursor gets the code  (Cursor didn't ask for it)
Claude gets nothing   (its callback was never invoked)
```

---

## Why This Doesn't Happen With a Single Client

When only Cursor is used:

- Cursor registers `redirect_uri=https://cursor.example.com/oauth/callback`
- Worker whitelists that same URL
- Every auth flow goes back to Cursor's callback
- Everything works

The bug is **invisible with one client** and only surfaces when a second
client (Claude) with a different callback URL enters the picture.

---

## How Your Current Code is Involved

### `__main__.py` — Discovery document

```python
# __main__.py line 80
return JSONResponse({
    "issuer": settings.mcp_public_url,
    "authorization_endpoint": f"{worker}/authorize",
    "token_endpoint":         f"{worker}/token",
    "registration_endpoint":  f"{worker}/register",
    ...
})
```

Both Claude and Cursor discover the **same Worker URL**. There is no per-client
routing here — both clients are pointed at the same `/authorize` endpoint.
If that endpoint doesn't properly respect each client's `redirect_uri`,
all clients collide.

### `config.py` — Single `oauth_worker_url`

```python
# config.py line 97
oauth_worker_url: str = ""
```

There is one Worker URL for all clients. The Worker itself must handle
multi-client `redirect_uri` isolation. If it doesn't, this is where the
fix needs to land.

### `middleware.py` — Single `CF_AUD_TAG`

```python
# config.py lines 84-89
cf_aud_tag: str = ""

@property
def cf_aud_tags(self) -> list[str]:
    return [t.strip() for t in self.cf_aud_tag.split(",") if t.strip()]
```

The MCP server validates the CF JWT's `aud` claim against `CF_AUD_TAG`. If
Cursor and Claude are issued tokens with **different audience tags** (because
they go through different CF Access apps), but the server only accepts one
`CF_AUD_TAG`, one of them will always get a 401. The server already supports
**comma-separated multiple AUD tags** — this is the right place to add
Claude's AUD tag if needed.

---

## The Fix

### Option 1 — Fix the CF Worker to Respect Per-Client `redirect_uri` (Recommended)

The Worker's `/authorize` and `/token` handlers must:

1. On `/register`: accept and store each client's `redirect_uris`.
2. On `/authorize`: use the `redirect_uri` from the request (not a hardcoded one).
   Validate it against the stored list for that `client_id`.
3. On `/token`: validate that the `redirect_uri` in the token exchange request
   matches exactly the one used in the `/authorize` request.

```
Before (broken):
  Worker /authorize always redirects to: https://cursor.example.com/oauth/callback

After (fixed):
  Worker /authorize reads redirect_uri from the request:
    Claude sends  → https://claude.ai/oauth/callback   → Worker uses this
    Cursor sends  → https://cursor.example.com/oauth/callback → Worker uses this
```

### Option 2 — Separate CF Access Apps per Client

Register two separate Cloudflare Access applications:
- **App A** for Cursor: `redirect_uri = https://cursor.example.com/oauth/callback`
- **App B** for Claude: `redirect_uri = https://claude.ai/oauth/callback`

Each app gets its own AUD tag. Add both AUD tags to `CF_AUD_TAG` in `.env`:

```env
CF_AUD_TAG=cursor-aud-tag-abc123,claude-aud-tag-def456
```

The MCP server already supports multiple AUD tags (comma-separated) — no
code change needed here.

### Option 3 — Use Separate Worker Deployments per Client

Deploy two CF Worker instances, one per client, each hardcoded to its
client's `redirect_uri`. Update the discovery doc to route clients to their
respective Workers. This is the most isolated but also the most overhead to
maintain.

---

## Recommended Immediate Fix

**Fix the CF Worker** to properly store and use per-client `redirect_uri`
(Option 1). This is the correct implementation of RFC 6749 (OAuth 2.0) and
RFC 7591 (Dynamic Client Registration) and is the standard way multi-client
OAuth servers work.

Additionally, add Claude's AUD tag to `CF_AUD_TAG` in `.env` if Claude and
Cursor are issued JWTs from separate CF Access apps:

```env
# .env
CF_AUD_TAG=your-cursor-aud-tag,your-claude-aud-tag
```

No changes are needed in the MCP server Python code — the multi-AUD support
is already implemented in `config.py`.

---

## Quick Debugging Checklist

When you see the callback landing on the wrong client:

- [ ] Check the CF Worker `/authorize` handler — is it using `redirect_uri`
      from the request or a hardcoded value?
- [ ] Check the CF Worker `/register` handler — is it storing each client's
      `redirect_uris` and associating them with the issued `client_id`?
- [ ] Check the CF Worker `/token` handler — is it validating that the
      `redirect_uri` matches the one from the original `/authorize` call?
- [ ] Check `CF_AUD_TAG` in `.env` — does it include the AUD tags for all
      clients that need access?
- [ ] Clear browser cookies for `mcp.example.com` before testing to rule out
      session bleed between clients.
