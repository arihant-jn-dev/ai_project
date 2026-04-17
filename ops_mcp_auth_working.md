# MCP Auth Working — Internal Deep Dive

How the MCP server processes every incoming request, validates the CF JWT, and why
the middleware logs fire multiple times for the same authenticated user.

---

## 1. What Happens on Every Single HTTP Request

The MCP server has **no concept of a session**. This is intentional:

```python
# server.py
mcp = FastMCP(
    settings.server_name,
    stateless_http=True,   # ← this is the key
)
```

`stateless_http=True` means the server treats every HTTP request as completely
independent. There is no login cookie, no server-side session store, no "remember
this user" mechanism. Every request must carry the token, and the token is validated
fresh every time.

So the middleware runs — and logs — **once per HTTP request**, not once per user.

---

## 2. How Many HTTP Requests Does One Tool Call Generate?

This is why you see multiple log lines. Cursor's MCP client (Streamable HTTP
transport, spec 2025-03-26) makes several independent HTTP requests during normal
operation:

| # | What Cursor sends | Triggers middleware? | Why |
|---|---|---|---|
| 1 | `POST /` — `initialize` | Yes | Cursor's handshake to discover server capabilities |
| 2 | `POST /` — `tools/list` | Yes | Cursor fetches the list of available tools |
| 3 | `POST /` — `tools/call` (tool A) | Yes | The actual tool call |
| 4 | `POST /` — `tools/call` (tool B) | Yes | If Cursor chains tool calls |
| 5 | `GET /` — SSE stream (optional) | Yes | Cursor may open a long-lived GET for server push events |

Every one of these is a separate TCP/HTTP request. Every one hits `CFAuthMiddleware.dispatch()`.
Every one logs `middleware:token-found` and `middleware:auth-ok` for the same user.

**This is correct and expected.** It is not a bug.

---

## 3. Step-by-Step: What Middleware Does on Each Request

Here is the exact code path through `middleware.py` on every request:

```
Incoming HTTP request (POST /)
          │
          ▼
CFAuthMiddleware.dispatch()
          │
          ├─ Is path in {/health, /healthz, /ready}?
          │      YES → bypass auth, pass straight through
          │      NO  → continue
          │
          ├─ Log request-received  (client IP, method, path)
          │
          ├─ _log_request_headers()
          │      Logs all headers; truncates Authorization + Cf-Access-Jwt-Assertion
          │
          ├─ _extract_token_with_source()
          │      1. Check Authorization: Bearer <token>   ← used first
          │      2. Check Cf-Access-Jwt-Assertion: <token> ← fallback
          │      Returns (token_string, "Authorization:Bearer") or (token_string, "Cf-Access-Jwt-Assertion") or (None, "none")
          │
          ├─ No token found?
          │      YES → return 401 + WWW-Authenticate header → STOP
          │
          ├─ Log middleware:token-found (source + token snippet + path)   ← THE LOG YOU SEE
          │
          ├─ validate_cf_token(token, settings)
          │      (details in section 4 below)
          │      Raises ValueError on any failure → 401 returned, STOP
          │
          ├─ Log middleware:auth-ok (email, sub, groups, path)
          │
          ├─ current_identity.set(CFIdentity)
          │      Stores the identity in a contextvar scoped to THIS request's async task
          │
          ├─ call_next(request)
          │      Hands control to the MCP layer (FastMCP tool handler)
          │      Tool handler runs entirely here — api.py calls happen here
          │
          └─ finally: current_identity.reset(ctx_token)
                 Clears the identity from the contextvar after the request is done
                 The next request starts with a clean slate
```

---

## 4. Step-by-Step: What `validate_cf_token` Does (auth.py)

This runs inside the middleware on every request:

```
validate_cf_token(token, settings)
          │
          ├─ Step 1: Decode JWT header (no signature check yet — just parsing)
          │      pyjwt.get_unverified_header(token)
          │      Extracts: kid="7cadf3c9bd4beedbfe6f0c0064e855ccfd5f11af..."
          │                alg="RS256"
          │      Fails fast on garbage tokens before any network call
          │
          ├─ Step 2: Fetch JWKS (Cloudflare's public keys)
          │      URL: https://custac.cloudflareaccess.com/cdn-cgi/access/certs
          │      CACHED in module-level dict for 1 hour (_JWKS_CACHE_TTL = 3600)
          │      On cache hit:  no network call — uses in-memory keys
          │      On cache miss: httpx.get(jwks_url) → stores result + timestamp
          │
          ├─ Step 3: Match kid → public key
          │      Loops through JWKS keys looking for key.kid == token.kid
          │      If no match → ValueError("No matching public key found for kid=...")
          │      On match → pyjwt.algorithms.RSAAlgorithm.from_jwk(key_data) → RSA public key object
          │
          ├─ Step 4: Verify signature + claims
          │      pyjwt.decode(
          │          token,
          │          public_key,          ← RSA public key from step 3
          │          algorithms=["RS256"],
          │          audience=settings.cf_aud_tags,   ← from CF_AUD_TAG in .env
          │          options={"verify_iss": False},
          │      )
          │      This checks:
          │        ✓ Signature valid (RS256 — cannot be forged without CF's private key)
          │        ✓ exp > now (token not expired)
          │        ✓ aud matches CF_AUD_TAG (right app, not a token from a different CF app)
          │        ✗ iss not checked (CF SaaS OIDC issuer URLs vary per app)
          │
          └─ Step 5: Extract claims → return CFIdentity
                 email  = payload["email"]    e.g. "arihant@customer-acquisition.co"
                 sub    = payload["sub"]      e.g. "31169eaf-f4a0-4af3-ad69-4ee80f807da1"
                 groups = payload["custom"]["groups"]  (empty [] if not configured in CF)
                 raw_token = the original token string (stored for forwarding to Laravel)
```

---

## 5. The JWKS Cache — Why Most Requests Don't Hit the Network

After the first request, JWKS keys are cached in memory:

```python
# auth.py
_jwks_cache: dict[str, Any] = {}
_jwks_cache_ts: float = 0.0
_JWKS_CACHE_TTL = 3600  # 1 hour
```

Timeline for 10 requests by the same user:

```
Request 1:  JWKS cache MISS  → fetches https://custac.cloudflareaccess.com/cdn-cgi/access/certs
Request 2:  JWKS cache HIT   → uses in-memory keys (age=2s, TTL=3600s)
Request 3:  JWKS cache HIT
...
Request N:  JWKS cache HIT   (as long as server has been running < 1 hour)

After 1 hour: cache expires → next request fetches fresh JWKS
```

Log lines to look for:
- `auth:jwks-cache-hit — 2 key(s) cached, age=14s (TTL=3600s)` → fast path, no network
- `auth:jwks-fetch — fetching fresh JWKS from https://...` → cache miss, network call

---

## 6. The `current_identity` Contextvar — How Identity Flows to Tool Handlers

This is the bridge between the middleware and the tool code:

```python
# auth.py
current_identity: contextvars.ContextVar[CFIdentity | None] = contextvars.ContextVar(
    "current_identity", default=None
)
```

A `ContextVar` is Python's async-safe version of thread-local storage. It is:
- **Set** by middleware at the start of each request: `current_identity.set(identity)`
- **Read** by any code in the same async task (same request): `current_identity.get()`
- **Reset** by middleware after the request finishes: `current_identity.reset(ctx_token)`
- **Invisible** to other concurrent requests — each request has its own copy

```
Request A (user: arihant)          Request B (user: bob)
─────────────────────────          ─────────────────────────
middleware sets:                   middleware sets:
  current_identity = arihant         current_identity = bob
          │                                  │
tool handler runs:                 tool handler runs:
  current_identity.get()             current_identity.get()
  → returns arihant's identity       → returns bob's identity
          │                                  │
middleware resets contextvar       middleware resets contextvar
```

No cross-request leakage is possible.

---

## 7. How the Token Gets From Middleware to api.py

Once middleware validates the token and sets `current_identity`, every tool handler
in `server.py` calls `_get_user_token()`:

```python
# server.py
def _get_user_token() -> str | None:
    identity = current_identity.get()  # reads contextvar set by middleware
    if identity:
        return identity.raw_token      # the original CF JWT string, unchanged
    return None                        # stdio mode fallback
```

`raw_token` is the **exact JWT string** that arrived in the `Authorization: Bearer`
header — nothing modified. It is passed into `api.py` as `user_token`.

`api.py._headers()` then builds the outgoing headers to Laravel:

```python
def _headers(settings, user_token=None):
    headers = {"Accept": "application/json", "Content-Type": "application/json"}
    if user_token:
        headers["Authorization"] = f"Bearer {user_token}"          # CF JWT forwarded
        headers["Cf-Access-Jwt-Assertion"] = user_token             # CF JWT again
    elif settings.api_token:
        headers["Authorization"] = f"Bearer {settings.api_token}"  # local dev fallback
    return headers
```

The token is **never modified, decoded, re-signed, or transformed** at any point
between arriving at the MCP server and being forwarded to Laravel.

---

## 8. Why You See Multiple `middleware:token-found` Logs for One User

Example: user asks "list all offers" → Cursor calls `list_offers` tool.

Here is every HTTP request Cursor actually sends, and what logs fire:

```
──────────────────────────────────────────────────────────────────────
HTTP 1: POST /  (method=initialize)
──────────────────────────────────────────────────────────────────────
middleware:request-received — POST / from client_ip=...
middleware:request-headers — POST / (all headers logged)
middleware:token-found — source=Authorization:Bearer token=eyJhbGc…r6gQ for POST /
auth:jwks-fetch — fetching fresh JWKS (first request only)
auth:jwt-validated-ok — email=arihant@... sub=31169eaf groups=[] exp=...
middleware:auth-ok — user=arihant@... sub=31169eaf groups=[] accessing POST /
[FastMCP handles initialize]
middleware:request-done — user=arihant@... POST / status=200 (12.4ms)

──────────────────────────────────────────────────────────────────────
HTTP 2: POST /  (method=tools/list)
──────────────────────────────────────────────────────────────────────
middleware:request-received — POST / from client_ip=...
middleware:token-found — source=Authorization:Bearer token=eyJhbGc…r6gQ for POST /
auth:jwks-cache-hit — 2 key(s) cached, age=1s (TTL=3600s)   ← no network call
auth:jwt-validated-ok — email=arihant@...
middleware:auth-ok — user=arihant@...
[FastMCP returns list of tools]
middleware:request-done — user=arihant@... POST / status=200 (3.1ms)

──────────────────────────────────────────────────────────────────────
HTTP 3: POST /  (method=tools/call → list_offers)
──────────────────────────────────────────────────────────────────────
middleware:request-received — POST / from client_ip=...
middleware:token-found — source=Authorization:Bearer token=eyJhbGc…r6gQ for POST /
auth:jwks-cache-hit — 2 key(s) cached, age=2s (TTL=3600s)
auth:jwt-validated-ok — email=arihant@...
middleware:auth-ok — user=arihant@...
tool:list_offers called — offer_type=None ...
user-token resolved: email=arihant@... token=eyJhbGc…
api:list-offers [a3f7c1b2] — params={} auth=user-jwt
api:request [a3f7c1b2] → GET https://.../get-offer-list
api:response [a3f7c1b2] ← 200 ... (143ms)
api:list-offers-done [a3f7c1b2] — success=True offer_count=47
middleware:request-done — user=arihant@... POST / status=200 (146ms)
```

**Bottom line: 3 `middleware:token-found` entries for one "list all offers" action.**
The JWT is validated 3 times (signature verified 3 times), but the JWKS is only
fetched from the network once (cached after that).

---

## 9. Full Request Lifecycle for One Tool Call (Single HTTP Request)

```
Cursor sends POST /
         │
         ▼
Starlette router:
  /.well-known/...  → _oauth_metadata (no auth, bypassed)
  /                 → CFAuthMiddleware (wraps FastMCP app)
         │
         ▼
CFAuthMiddleware.dispatch()
  1. Extract token from Authorization: Bearer header
  2. validate_cf_token() → CFIdentity(email, sub, groups, raw_token)
  3. current_identity.set(CFIdentity)  ← identity now available to all code below
  4. call_next(request) ──────────────────────────────────────────────┐
                                                                       │
         ▼  (inside call_next)                                         │
FastMCP receives the request                                           │
  Deserialises JSON-RPC body                                           │
  Routes to the right @mcp.tool() function                            │
         │                                                             │
         ▼                                                             │
Tool handler (e.g. list_offers in server.py)                          │
  _get_user_token()                                                    │
    └─ current_identity.get() → returns CFIdentity set in step 3      │
    └─ returns identity.raw_token (the CF JWT)                         │
  api.list_offers(settings, ..., user_token=<CF JWT>)                 │
         │                                                             │
         ▼                                                             │
api.py                                                                 │
  _log_request(req_id, "GET", url, headers, ...)                      │
  httpx.Client.get(url, headers={Authorization: Bearer <CF JWT>})     │
  → hits Laravel API                                                   │
  ← gets JSON response                                                 │
  _log_response(req_id, resp, ...)                                     │
  returns dict to tool handler                                         │
         │                                                             │
         ▼                                                             │
Tool handler returns result dict                                       │
FastMCP serialises to JSON-RPC response                               │
Returns HTTP 200 to Cursor ◄──────────────────────────────────────────┘
         │
         ▼
CFAuthMiddleware finally block:
  current_identity.reset(ctx_token)  ← wipes identity, clean slate for next request
```

---

## 10. What the Logs Mean — Quick Reference

| Log line | What it means |
|---|---|
| `middleware:request-received` | A new HTTP request arrived. Middleware started. |
| `middleware:request-headers` | All headers dumped (sensitive ones truncated). |
| `middleware:no-token` | No `Authorization` or `Cf-Access-Jwt-Assertion` header. Returns 401. |
| `middleware:token-found` | Token extracted. About to validate it. |
| `auth:jwks-cache-hit` | JWKS already in memory — no network call needed. Fast path. |
| `auth:jwks-fetch` | JWKS expired or cold start — fetching from CF. |
| `auth:jwt-header-decoded` | JWT header parsed. kid + alg extracted. |
| `auth:jwks-key-matched` | Found the right public key for this token's kid. |
| `auth:jwt-validated-ok` | Signature OK, not expired, audience matches. |
| `middleware:auth-ok` | Validation passed. Identity stored in contextvar. Request proceeding. |
| `user-token resolved` | Tool handler read the identity from contextvar. |
| `api:request [req_id]` | Outgoing HTTP request to Laravel. |
| `api:response [req_id]` | Laravel responded. Use same req_id to match to the request above. |
| `middleware:request-done` | Full request complete. Identity contextvar reset. |
| `middleware:token-invalid` | JWT failed validation (expired, wrong aud, bad sig). Returns 401. |

---

## 11. What Is NOT Stored Between Requests

| Thing | Stored? | Why |
|---|---|---|
| User identity (CFIdentity) | No — contextvar reset after each request | `stateless_http=True` + `finally: current_identity.reset()` |
| CF JWT token | No | Same as above |
| JWKS public keys | Yes — module-level dict, 1-hour TTL | Network optimisation only; safe to cache (public keys) |
| Laravel API token | No (not implemented yet) | See `AUTH_STRATEGY.md` Option A for future plan |
| MCP session | No | `stateless_http=True` disables FastMCP's session tracking |

---

## 12. Why `stateless_http=True` Was Chosen

The MCP server may run as multiple K8s pods. If session state were stored in memory
on pod A, a subsequent request from the same user landing on pod B would fail with
"Session not found".

`stateless_http=True` means the server carries zero per-user state. Any pod can
handle any request from any user at any time, as long as the request carries a valid
token. The JWKS cache is the only shared in-memory state — and it is safe to be
per-pod because it only contains Cloudflare's public keys (not secrets).

---

## 13. Summary

```
Every HTTP request from Cursor:
  │
  ├── Middleware runs (always)
  │     Extract token → validate JWT → set contextvar
  │
  ├── Tool handler runs (if auth passes)
  │     Read contextvar → get user token → call api.py
  │
  └── After request:
        Reset contextvar → no state persists

Multiple log lines per user = multiple HTTP requests per tool call
  (initialize + tools/list + tools/call + optional SSE GET)

JWKS validation is fast after the first request:
  public keys are cached in memory for 1 hour
  signature verification is CPU-only (no network)
```
