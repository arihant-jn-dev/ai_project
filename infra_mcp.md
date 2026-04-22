# MCP Server — Full Infrastructure Architecture

> **What this doc covers:** Every service in the stack, how machines connect, how auth works end-to-end, how the server is containerised and deployed on Kubernetes, and what Nginx will add for rate limiting and HTTPS.

---

## Table of Contents

1. [The Big Picture](#1-the-big-picture)
2. [Every Service Explained](#2-every-service-explained)
3. [How Machines Connect](#3-how-machines-connect)
4. [Login Flow — Step by Step](#4-login-flow--step-by-step)
5. [Tool Call Flow — Step by Step](#5-tool-call-flow--step-by-step)
6. [Dockerfile — How the Image Is Built](#6-dockerfile--how-the-image-is-built)
7. [Kubernetes — How the Pod Runs](#7-kubernetes--how-the-pod-runs)
8. [Nginx — What the SRE Will Add](#8-nginx--what-the-sre-will-add)
9. [Auth Deep Dive](#9-auth-deep-dive)
10. [URL and Endpoint Reference](#10-url-and-endpoint-reference)
11. [What the CF Worker Does NOT Do](#11-what-the-cf-worker-does-not-do)

---

## 1. The Big Picture

There are two completely separate paths in this system. It is important to keep them distinct in your head from the start.

**Path A — Login (happens once)**

```
Cursor IDE  →  CF Worker (workers.dev)  →  CF Access OIDC (custac.cloudflareaccess.com)
```

This path only runs when Cursor has no token — on first connect, or when the CF JWT expires after ~24 hours. After this path completes, the CF Worker is completely idle and is never contacted again until the next expiry.

**Path B — Every tool call (the main path)**

```
Cursor IDE  →  Twingate  →  Nginx (planned)  →  K8s MCP Pod (port 8080)  →  Laravel API
```

This path runs for every single tool call (`list_offers`, `get_offer`, `update_offer`, etc.). The CF Worker is not involved at all. The MCP server validates the CF JWT locally using cached public keys.

---

### Full Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  DEVELOPER MACHINE                                                            │
│                                                                               │
│  ┌────────────────────┐                                                       │
│  │    Cursor IDE       │                                                      │
│  │  (MCP client)       │                                                      │
│  └────────┬────────────┘                                                      │
│           │                                                                   │
└───────────┼───────────────────────────────────────────────────────────────────┘
            │
            │  TWO separate paths fork here
            │
            ├──── PATH A (login only, one-time) ────────────────────────────────►
            │                                                                    │
            │         ┌────────────────────────┐                                │
            │         │    CF Worker            │   workers.dev (Cloudflare edge)│
            │         │  /register              │                                │
            │         │  /authorize             │                                │
            │         │  /callback              │                                │
            │         │  /token                 │                                │
            │         └──────────┬─────────────┘                                │
            │                    │                                               │
            │         ┌──────────▼─────────────┐                                │
            │         │  CF Access OIDC         │   custac.cloudflareaccess.com  │
            │         │  (Google SSO login)     │                                │
            │         │  issues signed CF JWT   │                                │
            │         └────────────────────────┘                                │
            │                                                                    │
            │  ◄── CF JWT returned to Cursor. Login done. Worker exits stage. ──┘
            │
            │
            └──── PATH B (every tool call) ──────────────────────────────────►
                        │
                        ▼
            ┌───────────────────────────┐
            │         Twingate           │  zero-trust network access layer
            │  (cluster is unreachable   │  Developer must be connected to
            │   without this)            │  reach the K8s cluster at all
            └───────────┬───────────────┘
                        │
                        ▼
            ┌───────────────────────────┐
            │    Nginx  [PLANNED]        │  rate limiting + HTTPS termination
            │  port 443 → port 8080      │  your SRE configures rules here
            └───────────┬───────────────┘
                        │
                        ▼
            ┌───────────────────────────┐
            │   MCP Server (K8s Pod)     │  Python FastMCP server
            │      port 8080             │  Docker image in Kubernetes
            │                           │  validates CF JWT, runs tools
            └───────────┬───────────────┘
                        │                     ┌──────────────────────────┐
                        │  JWKS fetch          │  CF Access OIDC           │
                        │  (cached 1h) ───────►│  /cdn-cgi/access/certs   │
                        │                     └──────────────────────────┘
                        │
                        ▼
            ┌───────────────────────────┐
            │       Laravel API          │  your production backend
            │  validates CF JWT,         │
            │  applies RBAC, returns data│
            └───────────────────────────┘
```

---

## 2. Every Service Explained

### Cursor IDE

- **Runs on:** Developer's laptop
- **Role:** MCP client. Sends JSON-RPC tool calls over HTTP (POST `/`). Knows about MCP tools because the server told it at startup (`tools/list`).
- **OAuth behaviour:** Built-in OAuth 2.1 support. When it gets a `401` with a `WWW-Authenticate` header, it automatically opens the system browser for login and stores the resulting JWT internally.
- **Configuration:** `~/.cursor/mcp.json` — in production this is just `"url": "https://mcp-ops-interface.customer-acquisition.co"`. In local dev it's `"command"` + `"args"` + `"cwd"` to spawn a subprocess.

---

### Twingate

- **Runs on:** Network layer (zero-trust VPN equivalent)
- **Role:** The K8s cluster is not on the public internet. Twingate creates a secure tunnel so developer machines can reach the internal cluster network. If a developer is not connected to Twingate, the MCP server URL simply does not resolve — they cannot even start a connection, let alone authenticate.
- **Think of it as:** The outermost gate. No Twingate → no access, full stop.

---

### Nginx *(planned — your SRE is setting this up)*

- **Runs on:** In front of the K8s Pod, within the cluster or as an ingress
- **Role:**
  - **Rate limiting** — The SRE will define rules (e.g. max N requests per minute per IP, or per JWT). Requests over the limit get a `429 Too Many Requests`. This protects the MCP server and the Laravel API from abuse or runaway AI agents.
  - **HTTPS termination** — Nginx holds the TLS certificate and listens on port `443`. It strips TLS and forwards plain HTTP to the MCP server on port `8080` inside the cluster. The MCP server itself never handles TLS.
- **After Nginx is live:** Cursor connects via `https://mcp-ops-interface.customer-acquisition.co` (port 443). Before Nginx it connects via `http://` on port `8080` directly.

---

### MCP Server (Python, K8s Pod)

- **Runs on:** Kubernetes Pod, port `8080`
- **Role:** The core logic layer. Does several things:
  1. Serves the OAuth discovery endpoint (`/.well-known/oauth-authorization-server`) — public, no auth required
  2. Validates the CF JWT on every tool call via `CFAuthMiddleware`
  3. Runs tool handlers (`list_offers`, `get_offer`, `update_offer`, `confirm_offer_update`)
  4. Calls the Laravel API with the user's JWT on behalf of each tool call
- **Transport mode:** `MCP_TRANSPORT=sse` in production. Uses Streamable HTTP (POST `/` for tool calls, GET `/` for SSE stream). In local dev this is `stdio` instead.
- **Built from:** A Dockerfile in this repo. Deployed as a K8s Pod.

---

### CF Worker (OAuth shim)

- **Runs on:** Cloudflare's global edge network (`leadgen-mcp-oauth.tech-2e7.workers.dev`)
- **Role:** Handles the login ceremony **only**. Exposes four endpoints:
  - `POST /register` — dynamic client registration (Cursor registers itself)
  - `GET /authorize` — starts login, redirects to CF Access Google login
  - `GET /callback` — receives CF auth code, exchanges it for a signed CF JWT
  - `POST /token` — returns the CF JWT to Cursor as the `access_token`
- **After login:** Completely idle. Never involved in tool calls. Only called again when the JWT expires (~24h).
- **Implemented using:** `@cloudflare/workers-oauth-provider` library (TypeScript). Deployed via Wrangler CLI.

---

### CF Access OIDC

- **Runs on:** `custac.cloudflareaccess.com` (Cloudflare Zero Trust)
- **Role:** Cloudflare's identity provider. When a user hits the CF Worker's `/authorize`, the Worker redirects them here for login. CF Access:
  1. Shows the Google SSO login page
  2. Enforces the Access Policy (only `@customer-acquisition.co` emails allowed)
  3. Issues a signed CF JWT (`id_token`) after successful login
  4. Publishes JWKS public keys at `/cdn-cgi/access/certs` — the MCP server fetches these to verify JWT signatures
- **Think of it as:** Cloudflare-managed identity. Like Auth0 or Okta but operated by Cloudflare.

---

### Laravel API

- **Runs on:** Your production backend
- **Role:** The actual data layer. The MCP server forwards every tool call to it. Laravel validates the CF JWT independently (it also knows the public keys), applies role-based access control, and returns offer data.
- **Headers it receives from MCP server:**
  - `Authorization: Bearer <CF JWT>` — standard auth header
  - `Cf-Access-Jwt-Assertion: <CF JWT>` — Cloudflare-specific header that Laravel's CF middleware expects

---

## 3. How Machines Connect

### The network path in order

```
Developer laptop
    │
    │  Twingate client running on the laptop establishes a secure tunnel
    │
    ▼
Twingate network access layer
    │
    │  Routes traffic to the K8s cluster's internal network
    │
    ▼
K8s cluster (internal network, not on public internet)
    │
    │  [After Nginx is added]
    │  Nginx ingress — port 443 (HTTPS), terminates TLS, proxies to port 8080
    │
    ▼
MCP Server Pod — port 8080
```

### Why Twingate specifically

The K8s cluster runs in a private network. No port is exposed to the public internet. Twingate is a zero-trust access solution — instead of a traditional VPN that gives access to the whole network, it grants access to specific resources (in this case, the MCP server URL). If a developer's Twingate session expires, they cannot reach the server even if they have a valid CF JWT.

### The CF Worker lives outside this network

The CF Worker runs on Cloudflare's global edge — completely separate from your K8s cluster. It is publicly accessible by design (developers' browsers need to reach it during login). It has no access to your K8s cluster and never talks to it.

---

## 4. Login Flow — Step by Step

This flow happens once per user per day (CF JWTs expire after ~24 hours).

```
Step 1 — Cursor connects with no token
──────────────────────────────────────
Cursor sends:  POST https://mcp-ops-interface.customer-acquisition.co/
               (no Authorization header)

MCP server responds:
  HTTP 401 Unauthorized
  WWW-Authenticate: Bearer realm="leadgen-ops",
    resource_metadata="https://mcp-ops-interface.customer-acquisition.co
                        /.well-known/oauth-authorization-server"

The WWW-Authenticate header tells Cursor: "You need to auth — discover
how from this URL." This is the MCP OAuth spec (RFC 8414 adoption).


Step 2 — Cursor reads the OAuth discovery document
───────────────────────────────────────────────────
Cursor sends:  GET /.well-known/oauth-authorization-server

MCP server responds (served by __main__.py, outside auth middleware):
  {
    "issuer":                "https://mcp-ops-interface.customer-acquisition.co",
    "authorization_endpoint":"https://leadgen-mcp-oauth.tech-2e7.workers.dev/authorize",
    "token_endpoint":        "https://leadgen-mcp-oauth.tech-2e7.workers.dev/token",
    "registration_endpoint": "https://leadgen-mcp-oauth.tech-2e7.workers.dev/register"
  }

Now Cursor knows the CF Worker's URLs.


Step 3 — Cursor registers itself with the CF Worker
────────────────────────────────────────────────────
Cursor sends:  POST https://leadgen-mcp-oauth.tech-2e7.workers.dev/register
               { "redirect_uris": ["http://localhost:<random-port>/callback"] }

CF Worker responds:
  { "client_id": "...", "client_secret": "..." }

Cursor stores these for the rest of the flow. The Worker stores
them in its KV namespace.


Step 4 — Cursor opens the system browser for login
────────────────────────────────────────────────────
Cursor opens the system browser (Chrome, Firefox, etc.) to:
  https://leadgen-mcp-oauth.tech-2e7.workers.dev/authorize
    ?client_id=<from step 3>
    &redirect_uri=http://localhost:<port>/callback
    &response_type=code
    &code_challenge=<S256 PKCE hash>
    &code_challenge_method=S256
    &state=<random>

Cursor also starts a temporary HTTP server on localhost:<port> to
catch the redirect back from the Worker after login.


Step 5 — CF Worker redirects to CF Access login
────────────────────────────────────────────────
The Worker receives /authorize, finds no CF auth code yet (Scenario A
in access-handler.ts), and redirects the browser to CF Access:

  https://custac.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/authorization
    ?client_id=<CF OIDC app client_id>
    &response_type=code
    &redirect_uri=https://leadgen-mcp-oauth.tech-2e7.workers.dev/callback
    &state=<base64-encoded original OAuth request info>
    &scope=openid email profile

The user sees the Cloudflare Access login page (Google SSO button).


Step 6 — User authenticates
────────────────────────────
User clicks "Sign in with Google" and picks their @customer-acquisition.co
account. CF Access evaluates the Allow policy — if the email is not on
the allowed list, access is denied here. The MCP server never sees this.

On success, CF Access issues an auth code and redirects the browser back
to the Worker's callback:
  GET https://leadgen-mcp-oauth.tech-2e7.workers.dev/callback?code=<CF_CODE>&state=...


Step 7 — Worker exchanges CF code for JWT
─────────────────────────────────────────
The Worker receives /callback (Scenario B in access-handler.ts):

  POST https://custac.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/token
  Body: grant_type=authorization_code, code=<CF_CODE>,
        client_id=..., client_secret=<ACCESS_CLIENT_SECRET>

CF Access responds with:
  {
    "id_token":     "<signed CF JWT>",   ← this is what we want
    "access_token": "...",
    "expires_in":   86400
  }

The id_token is the CF JWT. It contains:
  { "email": "user@customer-acquisition.co",
    "sub": "cf-user-uuid",
    "aud": ["<CF_AUD_TAG>"],
    "iss": "https://custac.cloudflareaccess.com",
    "exp": 1776488551 }

The Worker calls completeAuthorization(), which stores the JWT in its
KV namespace and redirects the browser to Cursor's localhost callback.


Step 8 — Cursor gets the JWT
─────────────────────────────
Browser redirects to: http://localhost:<port>/callback?code=<OAUTH_CODE>

Cursor's localhost server picks this up and calls:
  POST https://leadgen-mcp-oauth.tech-2e7.workers.dev/token
       { code: <OAUTH_CODE>, code_verifier: <PKCE verifier> }

CF Worker looks up the code in KV, verifies PKCE, and returns:
  { "access_token": "<CF JWT>", "token_type": "Bearer" }

Cursor stores the JWT. Browser tab closes.

THE CF WORKER IS NOW DONE. It will not be contacted again until
the JWT expires (~24h later) and the user needs to re-login.
```

---

## 5. Tool Call Flow — Step by Step

After login, every tool call follows this path. The CF Worker is not involved at all.

```
You type: "list all active hostpost offers"
                │
                ▼
Cursor AI reasons and decides to call list_offers(offer_type="hostpost")
                │
                ▼
Cursor sends:
  POST https://mcp-ops-interface.customer-acquisition.co/
  Authorization: Bearer <CF JWT>
  Content-Type: application/json
  Body: { "jsonrpc": "2.0", "id": 2, "method": "tools/call",
          "params": { "name": "list_offers",
                      "arguments": { "offer_type": "hostpost" } } }
                │
                ▼
── Twingate ───────────────────────────────────────────────────────────
Routes the request to the K8s cluster's internal network.
Without Twingate connected: connection refused before reaching Nginx.
                │
                ▼
── Nginx [PLANNED] ────────────────────────────────────────────────────
Checks rate limiting rules. If over the limit → returns 429, done.
Strips TLS (HTTPS → HTTP). Forwards to MCP server on port 8080.
                │
                ▼
── CFAuthMiddleware (middleware.py) ────────────────────────────────────
1. Checks path: /health, /healthz, /ready → bypass, no auth needed
2. Extracts JWT from Authorization: Bearer header
3. If no JWT → return 401 + WWW-Authenticate (triggers re-login)
4. validate_cf_token(jwt, settings):
   a. Decode JWT header → extract kid (key ID)
   b. _get_jwks() → from 1-hour in-memory cache (or fetch from CF)
   c. Find public key matching kid
   d. pyjwt.decode(): verify RS256 signature + exp + aud (CF_AUD_TAG)
   e. Extract email, sub, groups → CFIdentity
5. current_identity.set(CFIdentity(...)) — scoped to this async request
6. Pass request to FastMCP handler
                │
                ▼
── server.py: list_offers() tool handler ──────────────────────────────
1. _get_user_creds():
   - current_identity.get() → has email, sub, raw_token
   - token_store.get(sub) → check cached Laravel Passport token
   - Cache miss → api.laravel_login() → POST /api/auth/login with CF JWT
     → Laravel returns Passport token → cache it
2. api.list_offers(settings, user_token=passport_token, cf_jwt=raw_token)
                │
                ▼
── api.py: HTTP call to Laravel ────────────────────────────────────────
GET https://your-laravel-api.com/api/admin/offer/get-offer-list
    ?offerType=hostpost
Headers:
  Authorization: Bearer <CF JWT>
  Cf-Access-Jwt-Assertion: <CF JWT>
                │
                ▼
── Laravel API ─────────────────────────────────────────────────────────
Validates CF JWT. Applies RBAC. Returns offer list JSON.
                │
                ▼
Response flows back: api.py → server.py → JSON-RPC response → Cursor
                │
                ▼
Cursor receives:
  { "jsonrpc": "2.0", "id": 2,
    "result": { "content": [{ "type": "text", "text": "{ ... offer data ... }" }] } }

AI presents the results to you in natural language.
```

---

## 6. Dockerfile — How the Image Is Built

The `Dockerfile` in the repo root builds the MCP server image.

```dockerfile
FROM python:3.12-slim AS deps    # lightweight base image

WORKDIR /app
COPY requirements.txt ./
RUN useradd --create-home --shell /bin/bash appuser
RUN pip install --no-cache-dir -r requirements.txt   # dependencies layer (cached)

COPY mcp_server/ ./mcp_server/   # code layer (only re-runs on code changes)

USER appuser       # never run as root
EXPOSE 8080        # the port uvicorn binds to

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/sse')" || exit 1

CMD ["python", "-m", "mcp_server"]   # starts the server
```

### Why this structure matters

**Two-layer caching:** Dependencies (`requirements.txt`) and code (`mcp_server/`) are in separate `COPY` + `RUN` steps. Docker caches each layer independently. If you only change Python code, the `pip install` layer is already cached — the rebuild takes seconds, not minutes.

**Non-root user:** The container runs as `appuser`, not root. This is a security best practice for K8s workloads.

**Health check in the Dockerfile:** The `HEALTHCHECK` instruction tells Docker (and K8s) how to probe the container. It tries to reach `http://localhost:8080/sse` every 30 seconds. K8s uses this to restart unhealthy containers automatically.

**`python -m mcp_server`:** This runs `mcp_server/__main__.py`. In the container, `MCP_TRANSPORT=sse` is set via env var, so the server starts in HTTP mode (uvicorn on port 8080) rather than stdio mode.

---

## 7. Kubernetes — How the Pod Runs

### What K8s does for the MCP server

| K8s concern | What happens |
|---|---|
| Deployment | The Docker image is declared in a Deployment manifest. K8s pulls the image and starts the Pod. |
| Restarts | If the Pod crashes or the health check fails repeatedly, K8s replaces it automatically. |
| Configuration | Environment variables are injected from ConfigMap (non-secret config) and Secrets (sensitive values). The code never has credentials baked in. |
| Health probes | K8s hits `/health`, `/healthz`, and `/ready` to decide if the Pod is alive and ready for traffic. These paths are whitelisted in `CFAuthMiddleware` — they always pass with no auth check. |
| Networking | Other services in the cluster reach the Pod on port `8080`. Nginx (once added) will proxy external traffic to it. |

### Environment variables injected into the Pod

```
MCP_TRANSPORT=sse              → run in HTTP/SSE mode (not stdio)
AUTH_ENABLED=true              → enable CFAuthMiddleware
CF_JWKS_URL=https://custac.cloudflareaccess.com/cdn-cgi/access/certs
CF_AUD_TAG=<audience tag>      → JWT audience validation
CF_TEAM_DOMAIN=custac.cloudflareaccess.com
API_BASE_URL=https://your-laravel-api.com
OAUTH_WORKER_URL=https://leadgen-mcp-oauth.tech-2e7.workers.dev
MCP_PUBLIC_URL=https://mcp-ops-interface.customer-acquisition.co
LOG_LEVEL=INFO
```

### K8s health check paths and why they bypass auth

The server has three bypass paths in `CFAuthMiddleware`:
- `GET /health`
- `GET /healthz`
- `GET /ready`

K8s probes hit these paths from inside the cluster with no HTTP headers at all. If they had to pass JWT validation, K8s could never confirm the Pod is healthy — it would always get `401`. The bypass ensures probes always work regardless of auth state.

---

## 8. Nginx — What the SRE Will Add

Nginx will sit between Twingate and the MCP server Pod. Here is what changes:

### Before Nginx (current state)

```
Cursor → Twingate → MCP Server Pod:8080
                    (HTTP, no TLS, no rate limiting)
```

### After Nginx

```
Cursor → Twingate → Nginx:443 (HTTPS) → MCP Server Pod:8080 (HTTP)
                    rate limiting
                    TLS termination
                    → 429 if over limit
```

### Rate limiting

The SRE will define rules in the Nginx config, for example:

```nginx
limit_req_zone $binary_remote_addr zone=mcp_per_ip:10m rate=60r/m;

location / {
    limit_req zone=mcp_per_ip burst=20 nodelay;
    limit_req_status 429;
    proxy_pass http://mcp-server:8080;
}
```

This means: max 60 requests per minute per IP address, with a burst of 20. Requests over the limit get `429 Too Many Requests`. The MCP server never sees the over-limit requests.

Rules can also be per-token (using `$http_authorization` as the key) rather than per-IP, to apply per-user limits.

### HTTPS / TLS termination

Nginx holds the TLS certificate for `mcp-ops-interface.customer-acquisition.co`. It:
1. Listens on port `443` for HTTPS connections
2. Decrypts TLS
3. Forwards plain HTTP to the MCP server on port `8080` inside the cluster

The MCP server (uvicorn) never deals with certificates. From its perspective, every request arrives as plain HTTP from Nginx. This is standard practice — TLS termination at the ingress/load-balancer layer, plain HTTP inside the cluster.

---

## 9. Auth Deep Dive

### Layer 1 — CF JWT validation (inbound, every request)

```
JWT arrives at CFAuthMiddleware
        │
        ├─ 1. Decode JWT header (no network) → extract kid
        │
        ├─ 2. _get_jwks() → in-memory cache
        │       ├─ cache age < 1h → return cached keys (0ms, pure memory)
        │       └─ cache expired → fetch from custac.cloudflareaccess.com (50–200ms)
        │
        ├─ 3. Find public key matching kid
        │
        ├─ 4. pyjwt.decode():
        │       ├─ RS256 signature verification
        │       ├─ exp check (reject if expired)
        │       └─ aud check (must match CF_AUD_TAG)
        │
        └─ 5. current_identity.set(CFIdentity(email, sub, groups, raw_token))
               scoped to this request's async task — concurrent requests
               each get their own slot (ContextVar, not a global)
```

### Layer 2 — Laravel Passport token (outbound, per user)

After JWT validation passes, the MCP server needs to call Laravel on behalf of this user. It uses a second token:

```
tool handler calls _get_user_creds()
        │
        ├─ current_identity.get() → CFIdentity with sub (user UUID)
        │
        ├─ token_store.get(sub) → check in-memory cache keyed by CF sub
        │       ├─ Cache hit → return Passport token immediately (0 network calls)
        │       └─ Cache miss → api.laravel_login():
        │               POST /api/auth/login with CF JWT
        │               Laravel verifies CF JWT, returns Passport access_token
        │               Store in token_store with expiry
        │
        └─ Return (cf_jwt, passport_token) to tool handler
```

The 401 auto-recovery: if Laravel returns a `401` on any API call, the token is invalidated from the cache and the next call automatically re-logs in.

### Why two tokens?

| Token | Purpose | Validated by |
|---|---|---|
| CF JWT | Proves this is an authenticated Cloudflare user | CFAuthMiddleware (MCP server) via JWKS |
| Laravel Passport token | Proves this user is authorised in the Laravel application | Laravel itself |

The CF JWT is a Cloudflare-issued identity assertion. The Laravel Passport token is Laravel's own OAuth2 token — Laravel's APIs expect the Bearer token format it manages. The MCP server exchanges the CF JWT for a Laravel token once per session and caches it.

---

## 10. URL and Endpoint Reference

### MCP Server endpoints (`mcp-ops-interface.customer-acquisition.co`)

| Method | Path | Auth required? | Purpose |
|---|---|---|---|
| `GET` | `/.well-known/oauth-authorization-server` | No | OAuth discovery. Returns CF Worker's authorize/token/register URLs. Public — mounted outside auth middleware. |
| `POST` | `/` | Yes (CF JWT) | Every tool call. Streamable HTTP MCP endpoint. |
| `GET` | `/` | Yes (CF JWT) | SSE stream subscription (optional persistent connection). |
| `GET` | `/health` | No | K8s liveness probe. Auth bypass. |
| `GET` | `/healthz` | No | K8s liveness probe. Auth bypass. |
| `GET` | `/ready` | No | K8s readiness probe. Auth bypass. |

### CF Worker endpoints (`leadgen-mcp-oauth.tech-2e7.workers.dev`)

| Method | Path | Called by | Purpose |
|---|---|---|---|
| `POST` | `/register` | Cursor | Dynamic client registration. Returns `client_id` + `client_secret`. |
| `GET` | `/authorize` | Cursor (browser) | Starts login. Redirects to CF Access if no CF auth code present. |
| `GET` | `/callback` | CF Access (redirect) | Receives CF auth code. Exchanges for CF JWT. Redirects to Cursor. |
| `POST` | `/token` | Cursor | Returns CF JWT as `access_token`. |

### CF Access OIDC endpoints (`custac.cloudflareaccess.com`)

| Path | Called by | Purpose |
|---|---|---|
| `/cdn-cgi/access/sso/oidc/<client_id>/authorization` | CF Worker (redirect) | User-facing login page (Google SSO). |
| `/cdn-cgi/access/sso/oidc/<client_id>/token` | CF Worker (server-side) | Code-for-JWT exchange. |
| `/cdn-cgi/access/certs` | MCP server (`auth.py`) | JWKS public keys for JWT validation (cached 1h). |

---

## 11. What the CF Worker Does NOT Do

This is worth being explicit about because it is a common source of confusion.

```
WHAT THE CF WORKER DOES              WHAT THE CF WORKER DOES NOT DO
────────────────────────────────     ────────────────────────────────────────
/register  → client registration     ✗ Does NOT proxy tool calls
/authorize → start login flow        ✗ Does NOT see JSON-RPC messages
/callback  → exchange CF code        ✗ Does NOT validate JWTs on tool calls
/token     → return CF JWT           ✗ Is NOT in the tool call request path
                                     ✗ Does NOT store user data long-term
                                     ✗ Does NOT talk to the MCP server ever
```

After the user has a JWT (Step 8 of login), the Worker is completely idle until the JWT expires. Every tool call goes **directly** from Cursor to the MCP server. The Worker is never a proxy or a middleman for MCP traffic.

---

## Quick Reference — Local Dev vs Production

| Setting | Local dev (stdio) | Production (K8s) |
|---|---|---|
| `MCP_TRANSPORT` | `stdio` (default) | `sse` |
| `AUTH_ENABLED` | `false` | `true` |
| `API_BASE_URL` | `http://127.0.0.1:8000` | `https://your-laravel-api.com` |
| `API_TOKEN` | your personal dev token | empty (uses CF JWT flow) |
| `CF_JWKS_URL` | empty | `https://custac.cloudflareaccess.com/cdn-cgi/access/certs` |
| `CF_AUD_TAG` | empty | your CF Access app audience tag |
| Twingate needed? | No | Yes |
| Nginx | No | Yes (planned) |
| Docker | Not needed | Docker image in K8s |
| Cursor config | `command + args + cwd` | `url` only |
| CF Worker involved? | No | Yes (login only) |
