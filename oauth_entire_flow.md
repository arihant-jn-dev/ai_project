# OAuth Entire Flow — Leadgen MCP

## Overview: What Are All The Pieces?

| Component | URL / Location | Role |
|---|---|---|
| **MCP Server** | `https://mcp-stage.customer-acquisition.co` (K8s pod at `http://dashboard-mcp-stage-service.dashboard-api-staging:80`) | The actual tool server Cursor/Claude talks to for all tool calls |
| **CF Worker (OAuth Shim)** | `https://leadgen-mcp-oauth.tech-2e7.workers.dev` | Bridges CF Access authentication with MCP's OAuth 2.1 protocol. Used ONLY during login |
| **CF Access** | `custac.cloudflareaccess.com` | Does the actual identity check (Google SSO, email OTP, etc.) |
| **Backend API** | `http://dashboard-mcp-stage-service.dashboard-api-staging:80` | Laravel API that the MCP server calls after auth passes |
| **KV Store** | Cloudflare KV (`OAUTH_KV`) | Temporarily stores in-flight OAuth codes during login |

---

## Why Do We Need the CF Worker?

Claude.ai and Cursor follow the **OAuth 2.1 Authorization Code + PKCE** protocol for remote MCP authentication. They do not understand Cloudflare Access cookies or JWTs natively.

The problem: Cloudflare Access is great at protecting URLs and issuing JWTs, but it speaks "CF Access" not "OAuth 2.1". Claude/Cursor speak "OAuth 2.1" and cannot talk to CF Access directly.

The CF Worker is the **translation layer**:
- CF Access does login → issues a `CF_Authorization` cookie + JWT
- The Worker reads that cookie, wraps the JWT in an OAuth 2.1 Authorization Code response
- Claude/Cursor then gets the JWT as a Bearer token via the standard OAuth token endpoint

After login is complete, the Worker is never touched again. All tool calls go directly from Cursor/Claude → your MCP server.

---

## The Need for `mcp-stage.customer-acquisition.co`

Your MCP server runs inside Kubernetes at an internal address (`http://dashboard-mcp-stage-service.dashboard-api-staging:80`) which is not reachable from the internet or from Claude.ai.

`mcp-stage.customer-acquisition.co` is the **public-facing URL** for that K8s service. It is what Cursor/Claude adds as the connector URL. The K8s ingress routes traffic from this domain to the pod.

---

## The Full OAuth Flow — Step by Step

```
Claude.ai / Cursor
        │
        │  Step 1: User enters connector URL
        │  URL: https://mcp-stage.customer-acquisition.co
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  MCP Server  (K8s pod)                          │
  │  CFAuthMiddleware intercepts every request       │
  │                                                  │
  │  No Bearer token found → returns:               │
  │  HTTP 401                                        │
  │  WWW-Authenticate: Bearer realm="leadgen-ops",  │
  │    resource_metadata="https://mcp-stage.        │
  │    customer-acquisition.co/.well-known/          │
  │    oauth-authorization-server"                   │
  └─────────────────────────────────────────────────┘
        │
        │  Step 2: Cursor/Claude reads the WWW-Authenticate header
        │  and fetches the discovery document
        │
        ▼
  GET https://mcp-stage.customer-acquisition.co
      /.well-known/oauth-authorization-server
        │
        │  (This route is mounted OUTSIDE CFAuthMiddleware — always public)
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  MCP Server responds:                            │
  │  {                                               │
  │    "issuer":                "https://mcp-stage.…",│
  │    "authorization_endpoint":"https://leadgen-mcp-oauth…/authorize",│
  │    "token_endpoint":        "https://leadgen-mcp-oauth…/token",    │
  │    "registration_endpoint": "https://leadgen-mcp-oauth…/register", │
  │    "response_types_supported": ["code"],         │
  │    "grant_types_supported":    ["authorization_code"],│
  │    "code_challenge_methods_supported": ["S256"]  │
  │  }                                               │
  └─────────────────────────────────────────────────┘
        │
        │  Step 3: Cursor/Claude registers itself as an OAuth client
        │
        ▼
  POST https://leadgen-mcp-oauth.tech-2e7.workers.dev/register
  {client_name, redirect_uris, ...}

        │
        │  CF Worker (OAuthProvider) auto-issues a client_id and responds.
        │  /register has BYPASS policy in CF Access — no login needed.
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  CF Worker returns:                              │
  │  { client_id: "…", client_secret: "…", … }      │
  └─────────────────────────────────────────────────┘
        │
        │  Step 4: Cursor/Claude opens browser to /authorize
        │  with PKCE code_challenge, state, redirect_uri
        │
        ▼
  GET https://leadgen-mcp-oauth.tech-2e7.workers.dev/authorize
  ?response_type=code
  &client_id=…
  &redirect_uri=…
  &state=…
  &code_challenge=…
  &code_challenge_method=S256

        │
        │  /authorize is PROTECTED by CF Access.
        │  CF Access intercepts first — user has NOT logged in yet.
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  Cloudflare Access (custac.cloudflareaccess.com) │
  │                                                  │
  │  Checks: does browser have a valid CF session?  │
  │  No → redirects to login page                   │
  │  User completes login (Google SSO / email OTP)  │
  │  CF Access sets CF_Authorization cookie          │
  │  Redirects back to /authorize on the Worker     │
  └─────────────────────────────────────────────────┘
        │
        │  Step 5: Request arrives at CF Worker /authorize
        │  CF_Authorization cookie is now present
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  access-handler.ts (handleAccessRequest)         │
  │                                                  │
  │  1. Reads CF_Authorization cookie               │
  │  2. Base64-decodes JWT payload → extracts email │
  │  3. Calls completeAuthorization() with:         │
  │     - All OAuth PKCE params (from oauthReqInfo) │
  │     - userId = email from JWT                   │
  │     - metadata = { cfJwt: "<the full JWT>" }    │
  │                                                  │
  │  OAuthProvider library:                          │
  │  - Stores  code → { cfJwt, userId }  in KV     │
  │  - Redirects browser to redirect_uri?code=…    │
  └─────────────────────────────────────────────────┘
        │
        │  Browser is redirected back to Cursor/Claude
        │  with ?code=<authorization_code>&state=<state>
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  Cursor/Claude receives the authorization code  │
  └─────────────────────────────────────────────────┘
        │
        │  Step 6: Cursor/Claude exchanges code for token
        │
        ▼
  POST https://leadgen-mcp-oauth.tech-2e7.workers.dev/token
  {
    grant_type:    "authorization_code",
    code:          "…",
    code_verifier: "…",   ← PKCE verifier
    redirect_uri:  "…",
    client_id:     "…"
  }

        │
        │  /token has BYPASS policy in CF Access — no login needed.
        │  OAuthProvider looks up the code in KV.
        │  Verifies PKCE code_verifier against stored code_challenge.
        │  Returns the CF JWT that was stored in Step 5.
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  CF Worker responds:                             │
  │  {                                               │
  │    "access_token": "<CF JWT>",                   │
  │    "token_type":   "Bearer",                     │
  │    "expires_in":   …                             │
  │  }                                               │
  └─────────────────────────────────────────────────┘
        │
        │  Step 7: Cursor/Claude stores the CF JWT.
        │  From now on every request sends:
        │  Authorization: Bearer <CF JWT>
        │
        │  THE CF WORKER IS NEVER USED AGAIN.
        │
        ▼
  POST https://mcp-stage.customer-acquisition.co/
  Authorization: Bearer <CF JWT>
  Content-Type: application/json
  { jsonrpc MCP tool call }

        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  MCP Server — CFAuthMiddleware                   │
  │                                                  │
  │  _extract_token():                               │
  │    Reads Authorization: Bearer header           │
  │    Falls back to Cf-Access-Jwt-Assertion header │
  │                                                  │
  │  validate_cf_token():                            │
  │  1. Decode JWT header → get kid                 │
  │  2. Fetch JWKS from:                            │
  │     https://custac.cloudflareaccess.com/         │
  │     cdn-cgi/access/certs                        │
  │     (cached 1 hour in-memory)                   │
  │  3. Match kid → RSA public key                  │
  │  4. PyJWT.decode() verifies:                    │
  │     - RS256 signature                           │
  │     - Token not expired                         │
  │     - Audience matches CF_AUD_TAG               │
  │  5. Returns CFIdentity(email, sub, groups, raw) │
  │                                                  │
  │  current_identity.set(identity) ← contextvar    │
  │  Request passes to MCP layer                    │
  └─────────────────────────────────────────────────┘
        │
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  FastMCP Tool Handler (server.py)               │
  │                                                  │
  │  _get_user_token():                              │
  │    identity = current_identity.get()            │
  │    return identity.raw_token                    │
  │                                                  │
  │  api.list_offers(settings, user_token=<JWT>)    │
  │  api.get_offer(settings, user_token=<JWT>)      │
  │  api.update_offer_direct(...)                   │
  └─────────────────────────────────────────────────┘
        │
        │  CF JWT forwarded as-is to the backend
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │  Laravel Dashboard API                           │
  │  http://dashboard-mcp-stage-service.            │
  │  dashboard-api-staging:80                       │
  │                                                  │
  │  Receives the same CF JWT as a Bearer token.    │
  │  Can validate it against CF's JWKS to confirm   │
  │  the user's identity on its own side.           │
  └─────────────────────────────────────────────────┘
```

---

## Detailed Component Breakdown

### 1. MCP Server — `/__main__.py`

**Two transports:**

| Transport | When used | How |
|---|---|---|
| `stdio` | Local dev (default) | Cursor spawns the Python process, pipes JSON-RPC over stdin/stdout. No auth, no HTTP. |
| `sse` (Streamable HTTP) | Deployed / multi-user | Uvicorn runs an HTTP server. CFAuthMiddleware validates every request. |

**Route structure in SSE mode:**

```
Starlette app
├── GET /.well-known/oauth-authorization-server  ← NO auth (always public)
└── /* (everything else)                         ← CFAuthMiddleware wraps this
         └── mcp_app (FastMCP streamable HTTP)
```

The `/.well-known` route must be **outside** the middleware. If it were inside, Cursor/Claude would need a token to discover where to get a token — a chicken-and-egg problem.

---

### 2. `/.well-known/oauth-authorization-server` Response

Served by `_oauth_metadata()` in `__main__.py`.

```json
{
  "issuer":                       "https://mcp-stage.customer-acquisition.co",
  "authorization_endpoint":       "https://leadgen-mcp-oauth.tech-2e7.workers.dev/authorize",
  "token_endpoint":               "https://leadgen-mcp-oauth.tech-2e7.workers.dev/token",
  "registration_endpoint":        "https://leadgen-mcp-oauth.tech-2e7.workers.dev/register",
  "response_types_supported":     ["code"],
  "grant_types_supported":        ["authorization_code"],
  "code_challenge_methods_supported": ["S256"]
}
```

**Key point:** The `issuer` is the MCP server itself, but `authorization_endpoint`, `token_endpoint`, and `registration_endpoint` all point to the **CF Worker**. This is intentional — the MCP server is the resource server, the CF Worker is the authorization server.

---

### 3. CF Worker (`index.ts`) — The OAuth Shim

Built with `@cloudflare/workers-oauth-provider` which implements the full OAuth 2.1 server logic automatically.

**Routes and CF Access policies:**

| Route | Method | CF Access Policy | What happens |
|---|---|---|---|
| `/register` | POST | **Bypass** (everyone) | Cursor auto-registers as OAuth client. Handled by OAuthProvider automatically. |
| `/authorize` | GET | **Allow** (authenticated users only) | CF Access intercepts, forces login, sets cookie, then routes to `handleAccessRequest` |
| `/token` | POST | **Bypass** (everyone) | Cursor exchanges code → JWT. Handled by OAuthProvider, looks up code in KV. |
| `/mcp` | ANY | n/a | Returns 404 stub. Tool calls never go through the Worker. |

The `/register` and `/token` routes MUST be bypassed in CF Access because Cursor calls them programmatically (not via a browser) and cannot complete a CF Access login challenge.

**KV Store usage:** During the `/authorize` → `/token` exchange, OAuthProvider stores `{ code → { cfJwt, userId, pkce_challenge, redirect_uri, ... } }` in the `OAUTH_KV` namespace. The entry has a short TTL (minutes). Once the code is exchanged at `/token`, the entry is deleted.

---

### 4. `access-handler.ts` — The Core Bridge Logic

```
User hits /authorize (already logged in via CF Access)
         │
         ▼
handleAccessRequest()
         │
         ├── getCookieValue(headers, "CF_Authorization")
         │         │
         │         └── Returns the CF JWT (a signed RS256 token)
         │
         ├── extractEmail(cfJwt)
         │         │
         │         └── Decodes JWT payload (base64) → reads .email or .sub
         │             NOTE: No signature verification here — CF Access already verified it
         │
         └── completeAuthorization(request, env, {
                   ...oauthReqInfo,   ← PKCE params from the original /authorize request
                   userId:   email,
                   metadata: { cfJwt }
             })
                   │
                   └── OAuthProvider library stores code in KV
                       Redirects browser → redirect_uri?code=<code>&state=<state>
```

The `oauthReqInfo` object passed to `handleAccessRequest` is automatically populated by OAuthProvider from the `/authorize` query string: `redirect_uri`, `state`, `client_id`, `code_challenge`, `code_challenge_method`.

---

### 5. `middleware.py` — CFAuthMiddleware

Every HTTP request (except health checks and `/.well-known`) passes through this middleware.

**Token extraction order:**
1. `Authorization: Bearer <token>` — standard OAuth header. This is what Cursor/Claude sends after completing the OAuth flow.
2. `Cf-Access-Jwt-Assertion: <token>` — fallback. CF Access injects this header when a user browses to the MCP URL directly in a browser (not via Cursor).

**On missing token:**
Returns `HTTP 401` with a `WWW-Authenticate` header pointing back to the `/.well-known` URL. This triggers Cursor/Claude to restart the OAuth discovery flow.

**On invalid token:**
Returns `HTTP 401 { "error": "Invalid token: …" }`. The request never reaches the MCP layer.

**On valid token:**
Sets `current_identity` contextvar → request continues to FastMCP.

---

### 6. `auth.py` — JWT Validation

**JWKS caching:**
CF's public keys are fetched from `https://custac.cloudflareaccess.com/cdn-cgi/access/certs` and cached for 1 hour. This avoids making an HTTPS call on every tool invocation.

**Validation steps:**
1. `pyjwt.get_unverified_header(token)` — extracts `kid` (key ID) from JWT header. Catches garbage tokens early before any network call.
2. Fetch JWKS → find the public key matching `kid`.
3. `pyjwt.decode()` verifies:
   - RS256 signature (cryptographic proof the token was issued by CF)
   - `exp` claim (token not expired)
   - `aud` claim must match `CF_AUD_TAG` — this is the application audience tag, unique per CF Access Application. Every user who logs in gets the same `aud` value for the same app.
4. Returns `CFIdentity(email, sub, groups, raw_token)`.

**Why `verify_iss: False`?** CF SaaS OIDC issuer URLs vary per application and are not predictable. Skipping issuer verification is safe because the signature + audience check is sufficient.

---

### 7. `server.py` — MCP Tools and Token Forwarding

`_get_user_token()` reads the `current_identity` contextvar set by CFAuthMiddleware and returns the raw CF JWT string.

This token is passed as `user_token` to every `api.py` call. The Laravel backend receives the same CF JWT as `Authorization: Bearer <CF JWT>`. The backend can independently validate it if needed, or trust the MCP server's validation.

In `stdio` mode (local dev), `current_identity` is always `None`, so `user_token` returns `None`, and `api.py` falls back to the static `API_TOKEN` from `.env`.

---

## Environment Variables Mapping

| Variable | Where Used | Value (staging) |
|---|---|---|
| `MCP_PUBLIC_URL` | `/.well-known` issuer + `WWW-Authenticate` header | `https://mcp-stage.customer-acquisition.co` |
| `OAUTH_WORKER_URL` | `/.well-known` auth/token/register endpoints | `https://leadgen-mcp-oauth.tech-2e7.workers.dev` |
| `CF_JWKS_URL` | JWT signature verification | `https://custac.cloudflareaccess.com/cdn-cgi/access/certs` |
| `CF_AUD_TAG` | JWT audience validation | `53ab9a7d3d50521090b681db51507ab1679e63a19628ed4689f35c427c63caaa` |
| `CF_TEAM_DOMAIN` | CF team domain reference | `custac.cloudflareaccess.com` |
| `AUTH_ENABLED` | Enable CFAuthMiddleware in SSE mode | `true` |
| `MCP_TRANSPORT` | `stdio` or `sse` | `sse` in K8s |
| `API_BASE_URL` | Laravel backend | `http://dashboard-mcp-stage-service.dashboard-api-staging:80` |
| `COOKIE_ENCRYPTION_KEY` | CF Worker KV encryption | Set via `wrangler secret put` |

---

## What Happens After the First Login?

Once Cursor/Claude has the CF JWT:

1. It stores the token locally.
2. Every tool call is a direct HTTP request to `https://mcp-stage.customer-acquisition.co` with `Authorization: Bearer <CF JWT>`.
3. The CF Worker is never involved. No redirects, no cookies, no KV lookups.
4. The MCP server validates the JWT on every request (JWKS cached in memory).
5. When the token expires, Cursor/Claude will receive a `401` again and automatically restart the OAuth flow from Step 4 (it already has the `client_id` and the `/.well-known` discovery, so it goes straight to `/authorize`).

---

## Flow Comparison: First Login vs Subsequent Tool Calls

```
FIRST LOGIN (one time):
Cursor → MCP Server (401 + WWW-Authenticate)
Cursor → MCP Server /.well-known (get Worker URLs)
Cursor → CF Worker /register (get client_id)
Cursor → CF Worker /authorize → CF Access login → CF cookie
Cursor → CF Worker /token (exchange code → CF JWT)
Cursor stores JWT ✓

EVERY TOOL CALL (thereafter):
Cursor → MCP Server /  (Bearer: CF JWT)
MCP Server validates JWT against CF JWKS
MCP Server → Laravel API (Bearer: CF JWT)
Laravel API responds
MCP Server responds to Cursor
```

---

## Is the Implementation Correct?

Yes. The implementation correctly follows the pattern described in the doc you referenced:

| Doc Step | Implementation |
|---|---|
| Claude discovers OAuth endpoints from `/.well-known` | ✅ `_oauth_metadata()` in `__main__.py`, mounted outside auth middleware |
| MCP server returns `401 + WWW-Authenticate` | ✅ `CFAuthMiddleware.dispatch()` in `middleware.py` |
| Redirects user to CF Access login | ✅ CF Access policy on `/authorize` route of the Worker |
| CF Access issues signed JWT, shim returns as OAuth token | ✅ `access-handler.ts` reads `CF_Authorization` cookie → `completeAuthorization()` |
| Claude stores JWT, sends as Bearer token | ✅ Standard OAuth client behavior, handled by Cursor/Claude automatically |
| MCP server validates JWT against CF Access JWKS | ✅ `validate_cf_token()` in `auth.py` |
| Forwards same JWT to backend API | ✅ `user_token=_get_user_token()` passed to all `api.py` calls |

One subtle but correct design decision: The `/.well-known` `issuer` is set to the MCP server URL (`mcp-stage.customer-acquisition.co`), not the Worker URL. This is correct per RFC 8414 — the issuer is the resource server identity, while the authorization server endpoints (on the Worker) are referenced separately.
