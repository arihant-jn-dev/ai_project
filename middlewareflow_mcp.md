# MCP Server — Middleware & Auth Flow

> Full breakdown of how `CFAuthMiddleware` works, when it fires, how JWT
> validation happens, and exactly how many times the middleware is hit per
> user query.

---

## 1. Two Modes — Middleware Only Runs in One of Them

The server boots in one of two transport modes controlled by `MCP_TRANSPORT`:

```
MCP_TRANSPORT=stdio   (default — local Cursor dev)
MCP_TRANSPORT=sse     (deployed HTTP/SSE — shared server)
```

| | stdio | sse (HTTP) |
|---|---|---|
| How Cursor connects | Spawns a local subprocess, pipes JSON-RPC over stdin/stdout | HTTP POST + GET to a shared URL |
| Middleware active? | **NO** — zero HTTP, nothing to intercept | **YES — every request** |
| JWT validation? | Never | Every inbound request |
| Auth required? | No | Yes (when `AUTH_ENABLED=true`) |

**Right now, your local Cursor setup uses stdio.** The middleware code exists and is loaded, but it is never invoked. This doc explains what happens in the deployed SSE path.

---

## 2. Server Startup — How Middleware Gets Wired In

`__main__.py` wires everything up at boot:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Starlette app (outermost router)                                    │
│                                                                      │
│  Route: GET /.well-known/oauth-authorization-server ─────────────── │◄── PUBLIC (no auth)
│                                                                      │    Returns OAuth metadata
│  Mount: /  ──────────────────────────────────────────────────────── │
│    └── CFAuthMiddleware  ◄─────────────────── wraps everything below │
│          └── FastMCP ASGI app (POST /, GET /)                        │
│                └── MCP tool handlers                                 │
└──────────────────────────────────────────────────────────────────────┘
```

The key lines in `__main__.py`:

```python
mcp_app = mcp.streamable_http_app()          # raw FastMCP ASGI app

if settings.auth_enabled:
    mcp_app = CFAuthMiddleware(mcp_app, settings)  # wrap it

app = Starlette(routes=[
    Route("/.well-known/oauth-authorization-server", _oauth_metadata),  # outside middleware
    Mount("/", app=mcp_app),                                             # inside middleware
])
```

`/.well-known` is intentionally mounted **outside** the middleware because the
OAuth client discovery request must succeed before the user has any token.

---

## 3. Middleware — Request Lifecycle (Step by Step)

Every inbound HTTP request that isn't a health-check goes through this exact sequence in `CFAuthMiddleware.dispatch()`:

```
Incoming HTTP request
        │
        ▼
┌───────────────────────────────┐
│  Is path in {/health,         │  YES ──► call_next() immediately (bypass auth)
│  /healthz, /ready}?           │
└───────────────────────────────┘
        │ NO
        ▼
┌───────────────────────────────┐
│  Log request + all headers    │  (sensitive headers are truncated, not leaked)
└───────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│  _extract_token_with_source(request)                   │
│                                                        │
│  Priority 1: Authorization: Bearer <token>             │
│  Priority 2: Cf-Access-Jwt-Assertion: <token>          │
│                                                        │
│  Returns: (token, source) or (None, "none")            │
└────────────────────────────────────────────────────────┘
        │
        ├── token is None ──► return 401 JSON immediately
        │                     (with WWW-Authenticate header pointing to
        │                      /.well-known for OAuth discovery)
        │
        ▼ token found
┌────────────────────────────────────────────────────────┐
│  validate_cf_token(token, settings)   [auth.py]        │
│                                                        │
│  Step 1: Decode JWT header → extract kid + alg         │
│          (no network, fails fast on garbage input)     │
│                                                        │
│  Step 2: Fetch JWKS from Cloudflare                    │
│          GET https://<team>.cloudflareaccess.com/…     │
│          ── Cached in memory for 1 hour ──             │
│          (only a real network call on first request    │
│           or after the 1h TTL expires)                 │
│                                                        │
│  Step 3: Find the public key whose kid matches         │
│                                                        │
│  Step 4: pyjwt.decode() — verify:                      │
│          • RS256 signature                             │
│          • exp (not expired)                           │
│          • aud == CF_AUD_TAG                           │
│                                                        │
│  Step 5: Extract email, sub, groups → CFIdentity       │
└────────────────────────────────────────────────────────┘
        │
        ├── ValueError / Exception ──► return 401 JSON
        │
        ▼ identity validated
┌────────────────────────────────────────────────────────┐
│  current_identity.set(identity)                        │
│  (contextvar — scoped to this exact async task/request)│
└────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│  call_next(request)                                    │
│  → FastMCP handles the JSON-RPC tool call              │
│  → tool handler reads current_identity.get()           │
└────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│  Log: user + path + status + elapsed_ms                │
│  current_identity.reset(ctx_token)   ◄── cleanup       │
│  Return response to Cursor                             │
└────────────────────────────────────────────────────────┘
```

---

## 4. How Many Times Does Middleware Fire Per Query?

**Once per HTTP request. Each tool call = one HTTP POST = one middleware hit.**

In Streamable HTTP transport (MCP spec 2025-03-26), Cursor sends:

| HTTP request | Purpose | Middleware fires? |
|---|---|---|
| `GET /.well-known/oauth-authorization-server` | OAuth discovery | **NO** (outside middleware) |
| `POST /` | Every tool call (JSON-RPC) | **YES** |
| `GET /` | SSE stream subscription (if used) | **YES** |
| `GET /health` | K8s liveness probe | **NO** (health bypass) |

### Per-tool middleware hit count

| Your query | Tools called | HTTP requests | Middleware hits |
|---|---|---|---|
| "give me active offers" | `list_offers` | 1 POST | **1** |
| "give me details of offer X" | `get_offer` | 1 POST | **1** |
| "update bid to $5" | `update_offer` (preview) | 1 POST | **1** |
| "yes confirm" | `confirm_offer_update` (execute) | 1 POST | **1** |
| Full update flow (preview + confirm) | 2 tools | 2 POSTs | **2** |

> The middleware does **not** fire for the outbound API calls your server
> makes to the Laravel backend. Those are `httpx` calls made from `api.py`
> and go out over the internet — they never touch the middleware.

---

## 5. Two-Layer Auth: CF JWT + Laravel Passport

There are two separate authentication tokens in play after the middleware passes:

```
Cursor ──[CF JWT]──► CFAuthMiddleware ──► MCP tool handler
                                                │
                                                ▼
                                       _get_user_creds()
                                                │
                               ┌────────────────┴─────────────────┐
                               │  TokenStore (in-memory cache)    │
                               │  key: CF user sub (UUID)         │
                               │  value: Laravel Passport token   │
                               └────────────────┬─────────────────┘
                                       │                  │
                                  cache HIT          cache MISS
                                       │                  │
                                       │         api.laravel_login()
                                       │         (POST to Laravel with CF JWT)
                                       │         → returns Passport token
                                       │         → store in TokenStore
                                       │                  │
                                       └──────────┬───────┘
                                                  ▼
                                      api.py calls Laravel API
                                      with Passport token in header
```

| Token | What it is | Who validates it | Cached? |
|---|---|---|---|
| CF Access JWT | Cloudflare-issued RS256 JWT | `CFAuthMiddleware` via `validate_cf_token()` | JWKS cached 1h |
| Laravel Passport token | Laravel OAuth2 bearer token | Laravel backend | Per-user, in `TokenStore` |

### Laravel token cache behaviour

- **Cache hit**: tool call uses the stored Passport token immediately (no extra network call)
- **Cache miss**: one `POST /login` to Laravel to exchange the CF JWT for a Passport token, then cached
- **401 from Laravel**: cache is invalidated (`_invalidate_on_auth_error`), next call re-logs in
- **Token expiry**: `TokenStore.get()` evicts tokens within 30s of their `exp` so a nearly-expired token is never returned

---

## 6. Identity Propagation via ContextVar

`current_identity` is a Python `contextvars.ContextVar`. It bridges the middleware and the tool handlers without any globals or thread-locals:

```
CFAuthMiddleware.dispatch()
    current_identity.set(CFIdentity(email, sub, groups, raw_token))
            │
            │  ← request is now "inside" the contextvar scope
            ▼
    FastMCP routes to tool handler (e.g. list_offers)
            │
            ▼
    server.py: _get_user_creds()
        identity = current_identity.get()   ← reads the value set above
            │
            ▼
    server.py: _IdentityFilter.filter()
        identity = current_identity.get()   ← also read by log filter
        injects user=email sub=... into every log line
            │
            ▼
    CFAuthMiddleware.dispatch() finally:
        current_identity.reset(ctx_token)   ← cleaned up after response sent
```

Because `contextvars` is async-safe, concurrent requests each see their own
identity value — no cross-request bleed even with many simultaneous users.

---

## 7. JWKS Cache — When Does It Hit the Network?

```
First request of the day
    └── _get_jwks() called
    └── _jwks_cache is empty → fetch from CF (network call, ~50–200ms)
    └── cache stored with timestamp

Requests 2 through N (within 1 hour)
    └── _get_jwks() called
    └── cache age < 3600s → return in-memory keys instantly (0ms)

After 1 hour
    └── next request triggers a fresh fetch
    └── new keys stored, timestamp reset
```

This means JWT signature verification is effectively **free** (pure CPU,
no I/O) for the vast majority of requests.

---

## 8. Full End-to-End Flow — "give me active offers" Query

```
You (in Cursor)
    │  types: "give me active offers count with their type"
    ▼
Cursor (MCP client)
    │  POST / HTTP/1.1
    │  Authorization: Bearer <CF Access JWT>
    │  Body: { "method": "tools/call", "params": { "name": "list_offers", ... } }
    ▼
CFAuthMiddleware.dispatch()
    │  1. Not a health path → proceed
    │  2. Log headers (tokens truncated)
    │  3. Extract token from Authorization: Bearer header
    │  4. validate_cf_token():
    │       a. Decode JWT header → kid
    │       b. _get_jwks() → from cache (or CF network)
    │       c. Match kid → RSA public key
    │       d. pyjwt.decode() → verify sig + exp + aud
    │       e. Extract email, sub, groups
    │  5. current_identity.set(CFIdentity(...))
    │  6. call_next(request) →
    ▼
FastMCP router
    │  Parses JSON-RPC → dispatches to list_offers()
    ▼
list_offers() in server.py
    │  _get_user_creds():
    │       a. current_identity.get() → has identity ✓
    │       b. TokenStore.get(sub) → cache hit? return Passport token
    │            └── cache miss? api.laravel_login() → POST /api/login to Laravel
    │  api.list_offers(settings, ..., user_token=..., cf_jwt=...)
    │       └── httpx POST to Laravel /api/offers
    │  Returns offer list
    ▼
CFAuthMiddleware.dispatch()
    │  Log: user=you@company.com POST / status=200 (142ms)
    │  current_identity.reset()
    ▼
Cursor receives JSON-RPC response
    ▼
You see the result
```

**Middleware fired: 1 time.**
**JWT validated: 1 time.**
**JWKS network call: 0 times** (if within the 1h cache window).

---

## 9. Deep Dive — How `current_identity` ContextVar Works

### The Problem It Solves

Your MCP server handles multiple users **at the same time** in the same Python process.
If you used a plain global variable to store "the current user", concurrent requests
would overwrite each other:

```python
# WRONG — shared global, breaks with concurrent requests
current_user = None

# Alice's request sets it
current_user = "alice@company.com"

# Bob's request immediately overwrites it
current_user = "bob@company.com"

# Alice's tool handler now reads "bob@company.com"  ❌
```

A `ContextVar` solves this by giving **each concurrent async task its own private copy**
of the variable. Same name, but isolated per request.

---

### What It Looks Like in Memory

When Alice and Bob call a tool at the same time:

```
current_identity  →  [slot for Alice's task]  =  CFIdentity("alice@company.com")
                     [slot for Bob's task]    =  CFIdentity("bob@company.com")

Alice's code: current_identity.get()  →  alice's identity  ✓
Bob's code:   current_identity.get()  →  bob's identity    ✓
```

No cross-contamination. Each async task is isolated automatically.

---

### Three Places It's Used in Your Code

**1. Defined once — `auth.py` line 68**

```python
current_identity: contextvars.ContextVar[CFIdentity | None] = contextvars.ContextVar(
    "current_identity", default=None
)
```

Creates the slot. Default is `None` for everyone until middleware sets it.

---

**2. SET by middleware — `middleware.py` line 131**

```python
ctx_token = current_identity.set(identity)   # set for THIS request's task only
try:
    response = await call_next(request)       # tool handler runs here and reads it
finally:
    current_identity.reset(ctx_token)         # cleanup — restore to None after response
```

`ctx_token` is a reset handle (not a JWT). The `finally` block guarantees cleanup
even if the tool crashes mid-execution.

---

**3. READ by tool handlers and the log filter — `server.py`**

```python
# _get_user_creds() — decides which API token to use
identity = current_identity.get()   # reads value set by middleware above

# _IdentityFilter — injects user into every log line automatically
identity = current_identity.get()
record.user = f"{identity.email} sub={identity.sub[:8]}"
```

Neither function receives `identity` as an argument. The middleware and tool handlers
communicate **through the contextvar** — completely decoupled.

---

### Concurrent Requests Side by Side

```
Alice's Request                        Bob's Request
──────────────────────────             ──────────────────────────
Middleware validates alice's JWT       Middleware validates bob's JWT
current_identity.set(alice_identity)   current_identity.set(bob_identity)
        │                                      │
        ▼                                      ▼
  list_offers() runs                    list_offers() runs
  current_identity.get()                current_identity.get()
  → alice_identity  ✓                   → bob_identity  ✓
        │                                      │
        ▼                                      ▼
  Log: [user=alice@...]                 Log: [user=bob@...]
        │                                      │
        ▼                                      ▼
  current_identity.reset() → None       current_identity.reset() → None
```

Both run concurrently. Both see only their own identity. Neither interferes with the other.

---

### Why Not Just Pass Identity as a Function Argument?

You could write `list_offers(identity=...)` — but the ContextVar approach wins because:

| Approach | Problem |
|---|---|
| Pass as argument | Every function in the chain needs the extra param: `list_offers` → `_get_user_creds` → `api.list_offers` → ... |
| Global variable | Race condition with concurrent requests |
| `threading.local` | Only works for threads, not `async/await` |
| **ContextVar** | Set once in middleware, readable anywhere in the same task, async-safe |

The `_IdentityFilter` is the best example — the logging system has no knowledge of
your request objects, but `current_identity.get()` lets it inject the right user into
every log line with zero plumbing.

---

### stdio Mode — Why It Returns `None`

In your local Cursor setup (stdio), the middleware never runs, so `current_identity`
stays at its default of `None`. That's why `_get_user_creds()` checks:

```python
if identity is None:
    return None, None   # fall back to static API_TOKEN from .env
```

The same code handles both modes — no `if transport == "sse"` checks scattered everywhere.

---

## 10. What Middleware Does NOT Do

| Concern | Handled by |
|---|---|
| Rate limiting | Not implemented (would need Redis or CF rules) |
| Role-based access control | Not implemented (groups are available in CFIdentity but not checked) |
| Per-tool authorization | Not implemented |
| Request body validation | FastMCP / Pydantic |
| Outbound API auth (Laravel) | `_get_user_creds()` + `TokenStore` in server.py |
| stdio transport auth | Nothing — stdio is inherently local/trusted |

---

## 10. Quick Reference — Files Involved

| File | Role |
|---|---|
| `__main__.py` | Boots server, wires middleware into the ASGI stack |
| `middleware.py` | `CFAuthMiddleware` — intercepts every HTTP request, validates CF JWT |
| `auth.py` | `validate_cf_token()`, JWKS cache, `CFIdentity`, `current_identity` contextvar |
| `server.py` | MCP tool handlers, `_get_user_creds()`, `_IdentityFilter` log injector |
| `token_store.py` | In-memory Laravel Passport token cache (keyed by CF sub) |
| `config.py` | `AUTH_ENABLED`, `CF_JWKS_URL`, `CF_AUD_TAG` and all other settings |
| `api.py` | Outbound HTTP calls to Laravel (never touches the middleware) |
