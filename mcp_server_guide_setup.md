# Leadgen Ops MCP Server — Complete Guide

## Table of Contents

1. [What is MCP and Why We Built This](#1-what-is-mcp-and-why-we-built-this)
2. [JSON-RPC, stdin, and SSE — The Three Core Concepts](#2-json-rpc-stdin-and-sse--the-three-core-concepts)
3. [Two Transport Modes: Local vs Production](#3-two-transport-modes-local-vs-production)
4. [Full Request Flow — How Everything Connects](#4-full-request-flow--how-everything-connects)
5. [Module Reference — Every File Explained](#5-module-reference--every-file-explained)
6. [Local Development Setup](#6-local-development-setup)
7. [Debugging Lessons (What We Learned)](#7-debugging-lessons-what-we-learned)

---

## 1. What is MCP and Why We Built This

**MCP (Model Context Protocol)** is an open standard that lets AI assistants (like Cursor, Claude) call your own backend tools. Instead of the AI guessing or hallucinating offer data, it calls your actual Laravel API through this server and gets real data back.

```
Without MCP:
  You → Cursor → AI guesses the offer bid

With MCP:
  You → Cursor → MCP Server → Laravel API → Real offer data → Cursor → Answer
```

### Why we need it

- AI can fetch live offer details, list offers, check bids — all from real data
- AI can update offers, save drafts, promote changes — with safety routing built in
- All without you having to copy-paste JSON from the dashboard manually

---

## 2. JSON-RPC, stdin, and SSE — The Three Core Concepts

These three things work together to make MCP function. They answer three different questions:

- **JSON-RPC** → what do the messages look like?
- **stdin/stdout** → how do messages travel in local dev?
- **SSE/HTTP** → how do messages travel in production?

---

### 2a. JSON-RPC — The Language (always used)

JSON-RPC is just an **agreed format for messages**. It has nothing to do with HOW messages travel — only what they look like.

```
Every request must have:
  jsonrpc → always "2.0" (protocol version)
  id      → a number to match each request with its response
  method  → what you want to call
  params  → arguments to pass

Every response must have:
  jsonrpc → "2.0"
  id      → same number as the request it answers
  result  → the data back
```

Think of it like a **phone call script** — you always introduce yourself, state what you want, and the other person gives you exactly what you asked for. The `id` is how you track which answer belongs to which question when many calls happen at once.

**In our project — when Cursor calls `list_offers`:**

Cursor sends this:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "list_offers",
    "arguments": { "offer_type": "hostpost" }
  }
}
```

Our server responds with this:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"success\": true, \"response\": [{\"offerName\": \"My Offer\", ...}]}"
      }
    ]
  }
}
```

**This format is identical whether running locally or in production.** Only the pipe it travels through changes.

---

### 2b. stdin/stdout — The Pipe (local dev transport)

`stdin` and `stdout` are channels every process has by default on any operating system. Like two water pipes permanently attached to every program:

```
stdin  → water flows IN  to your program   (another program writes here)
stdout → water flows OUT of your program   (another program reads from here)
stderr → a separate pipe just for logs     (never touches the main flow)
```

When Cursor launches your MCP server, it **connects its own pipe** to the Python process:

```
Cursor process                         Python process (mcp_server)
      │                                        │
      │─── writes JSON-RPC ──────────────► stdin
      │                                        │ (server reads, runs tool, responds)
      │◄── reads JSON-RPC ─────────────── stdout
      │
      stderr ──────────────────────────────► your terminal logs (Cursor ignores this)
```

This is the same as piping commands in your terminal:
```bash
echo "hello" | grep "hello"
#       ↑                ↑
#  writes to stdout    reads from stdin
```

Cursor does the exact same thing — it pipes JSON-RPC messages into the Python process and reads the responses back out.

**In our project — where this is set up:**

```python
# mcp_server/__main__.py

# All logging goes to stderr — never stdout
logging.basicConfig(stream=sys.stderr, ...)

# stdio mode: read from stdin, write to stdout
mcp.run(transport="stdio")
```

**Why logging MUST go to stderr and never stdout:**

If any log line accidentally goes to stdout, Cursor receives:
```
2026-04-19 16:00:33 INFO startup...    ← NOT valid JSON, breaks the parser
{"jsonrpc": "2.0", ...}                ← valid JSON
```
Cursor tries to parse the log line as JSON → fails → entire connection breaks. stderr is a completely separate pipe that Cursor ignores, so logs go there safely.

**Why stdin/stdout works great for local dev:**
- No network port needed
- Cursor manages the process (starts it, kills it automatically)
- One Python process per developer, completely isolated
- Zero infrastructure required

---

### 2c. SSE / Streamable HTTP — The Network Transport (production)

SSE = **Server-Sent Events**. It's an HTTP technique where the server keeps a connection open and streams messages back to the client as they arrive — instead of closing after every response.

**Regular HTTP (closes after each response):**
```
Client → POST /  (sends JSON-RPC request)
Server → response  ← connection closes
Client → POST /  (sends next request)
Server → response  ← connection closes again
```

**SSE (keeps connection open for streaming):**
```
Client → GET /  (opens a persistent stream)
Server → stays connected...
Server → ...pushes message 1 when ready...
Server → ...pushes message 2 when ready...
Server → ...pushes message 3 when ready...   ← connection stays open
```

In MCP's Streamable HTTP mode, both are used together:
- `POST /` → client sends a JSON-RPC request
- `GET /`  → server streams JSON-RPC responses back

**In our project — production flow:**

```
Any developer's Cursor
      │
      │  POST / with JSON-RPC request body
      ▼
Cloudflare Access (validates CF JWT — rejects unauthenticated requests)
      │
      ▼
Our MCP HTTP server (uvicorn on port 8080)
      │
      ├── CFAuthMiddleware checks JWT → sets current_identity
      │
      ├── FastMCP Streamable HTTP handler receives the request
      │
      └── Streams JSON-RPC response back via SSE
```

**In our project — where this is set up:**

```python
# mcp_server/__main__.py

transport = os.environ.get("MCP_TRANSPORT", "stdio").lower()

if transport == "sse":
    mcp_starlette = mcp.streamable_http_app()   # FastMCP builds the ASGI app
    app = Starlette(routes=[...])               # wrap with auth middleware
    uvicorn.run(app, host=host, port=port)      # serve over HTTP
else:
    mcp.run(transport="stdio")                  # local: use stdin/stdout
```

**Why SSE/HTTP for production:**
- Many developers connect to ONE shared server (no per-person Python install)
- Runs in Docker / Kubernetes — always online
- Cloudflare Access sits in front for security
- Central logging — all tool calls visible in one place

---

### Summary — which transport does what

```
                    LOCAL (stdio)                PRODUCTION (SSE/HTTP)
                    ─────────────                ─────────────────────
Transport           stdin / stdout pipe          HTTP POST + SSE stream
Who starts server   Cursor (spawns subprocess)   Docker / K8s (always running)
Connection          Local process pipe           Network URL
Auth                None  (AUTH_ENABLED=false)   Cloudflare JWT (AUTH_ENABLED=true)
Multi-user          No (1 process per dev)        Yes (1 server, N users)
mcp.json entry      command + args + cwd          url only

JSON-RPC format     ─────────── IDENTICAL IN BOTH CASES ───────────
```

**The JSON-RPC messages never change — only the pipe they travel through changes.**

Where each is configured in `mcp.json`:

```json
// Local dev — Cursor spawns the process itself
{
  "ops-mcp-local": {
    "command": "/Library/Frameworks/Python.framework/Versions/3.12/bin/python3",
    "args": ["-m", "mcp_server"],
    "cwd": "/Users/arihant.ja/Desktop/ops-mcp/leadgen-dashboard-mcp"
  }
}

// Production — Cursor connects to a running HTTP server
{
  "leadgen-ops": {
    "url": "https://mcp.your-domain.com"
  }
}

---

## 3. Two Transport Modes: Local vs Production

### Mode A — stdio (local development)

```
Cursor IDE
  │
  │  spawns subprocess:
  │  python3 -m mcp_server
  │
  ▼
mcp_server/__main__.py
  │
  └── mcp.run(transport="stdio")
        │
        Cursor ←── JSON-RPC over stdin/stdout ──► Python process
```

**How Cursor knows to start it** — via `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "ops-mcp-local": {
      "command": "/Library/Frameworks/Python.framework/Versions/3.12/bin/python3",
      "args": ["-m", "mcp_server"],
      "cwd": "/Users/arihant.ja/Desktop/ops-mcp/leadgen-dashboard-mcp"
    }
  }
}
```

- `command` — which Python to use
- `args: ["-m", "mcp_server"]` — tells Python to run the `mcp_server` package (finds `mcp_server/__main__.py`)
- `cwd` — working directory; Python loads `.env` from here

**Key behaviour in stdio mode:**
- No network port is opened
- Cursor manages the process lifecycle (starts it, kills it)
- All `logging` goes to **stderr** so it never corrupts the JSON-RPC stream on stdout
- No auth middleware — `AUTH_ENABLED=false` is the default

---

### Mode B — HTTP/SSE (production/staging deployment)

```
User's Cursor
  │
  │  connects via URL
  ▼
Cloudflare Access (auth gate)
  │
  ▼
mcp_server HTTP server (uvicorn on port 8080)
  │
  ├── CFAuthMiddleware validates CF JWT
  │
  ├── /.well-known/oauth-authorization-server  (public, no auth)
  │
  └── MCP Streamable HTTP handler
        │
        └── same tools, same JSON-RPC messages
```

**How Cursor connects:**

```json
{
  "mcpServers": {
    "leadgen-ops": {
      "url": "http://your-server-ip:8080"
    }
  }
}
```

**Key differences from stdio:**
- Server runs continuously (via Docker / K8s)
- `AUTH_ENABLED=true` → `CFAuthMiddleware` validates every request
- Multiple users can connect simultaneously
- A Cloudflare Access OAuth Worker handles the login flow

---

## 4. Full Request Flow — How Everything Connects

### Local flow (stdio)

```
You: "list all hostpost offers"
  │
  ▼
Cursor AI reasons → decides to call list_offers(offer_type="hostpost")
  │
  ▼
Cursor writes JSON-RPC to Python process stdin
  │
  ▼
mcp_server/server.py: list_offers() tool handler runs
  │
  ├── _get_user_creds() → returns (None, None) in stdio/no-auth mode
  │
  ▼
mcp_server/api.py: list_offers() → httpx GET /api/admin/offer/get-offer-list?offerType=hostpost
  │
  ▼
Laravel API responds with offer list JSON
  │
  ▼
api.py returns dict to server.py
  │
  ▼
server.py writes JSON-RPC response to stdout
  │
  ▼
Cursor reads response → AI presents results to you
```

#### What actually travels through stdin → stdout

Here is the exact bytes flowing through the pipe at each step:

**Step 1 — Cursor first asks: "what tools do you have?" (happens once at startup)**

Cursor writes this to your Python process's **stdin**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

Your server replies on **stdout**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "list_offers",
        "description": "List all advertising offers. Optionally filter by offerType...",
        "inputSchema": {
          "type": "object",
          "properties": {
            "offer_type": { "type": "string" },
            "sub_type":   { "type": "string" }
          }
        }
      },
      { "name": "get_offer", ... },
      { "name": "update_offer", ... }
    ]
  }
}
```

Cursor caches this tool list → this is what shows up in **Settings → MCP**.

---

**Step 2 — You ask: "list all hostpost offers"**

Cursor AI decides to call `list_offers` and writes to **stdin**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "list_offers",
    "arguments": {
      "offer_type": "hostpost"
    }
  }
}
```

Your `list_offers()` handler in `server.py` runs → calls `api.list_offers()` → hits Laravel → gets data back.

Server writes to **stdout**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"success\": true, \"response\": [{\"offerName\": \"My Hostpost Offer\", \"offerUniqueId\": \"2bd5b9b9-...\", \"offerType\": \"hostpost\", \"status\": true}, ...]}"
      }
    ]
  }
}
```

Cursor reads this → the AI parses the `text` field → presents a human-readable answer to you.

---

**Step 3 — What happens if the tool fails**

If Laravel returns an error, your server still writes a valid JSON-RPC response — it never crashes the pipe:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"success\": false, \"error\": \"HTTP 401: Unauthorized\"}"
      }
    ]
  }
}
```

The AI reads this and tells you "the call failed with a 401 — check your API_TOKEN".

---

**Why logging must go to stderr (not stdout)**

If any `print()` or log line accidentally writes to stdout, Cursor receives:
```
2026-04-19 16:00:33 INFO startup...   ← NOT valid JSON
{"jsonrpc": "2.0", ...}               ← valid JSON
```
Cursor tries to parse the whole thing as JSON → fails → connection breaks.

That is why `__main__.py` routes all logging to `stderr`:
```python
logging.basicConfig(stream=sys.stderr, ...)
```
stderr goes to your terminal/log file. stdout is sacred — only JSON-RPC messages.

---

### update_offer routing logic

Not all updates go directly to live. The server has a safety router:

```
update_offer() called
  │
  ├── Fetch current live offer (api.get_offer)
  │
  ├── Apply changes → build activityLog
  │
  ├── offer_filter.check_draft_or_live(activityLog)
  │         │
  │         ├── Bid change < 30%  → DIRECT
  │         ├── Bid change ≥ 30%  → DRAFT
  │         ├── categoryId        → DRAFT  (always)
  │         ├── budget            → DRAFT  (always)
  │         └── name / type       → DIRECT (always)
  │
  ├── DIRECT path
  │     └── api.update_offer_direct()
  │           PUT /update-offer-complete/{type}/update  with saveType='direct'
  │           Backend uses request body immediately → goes live
  │
  └── DRAFT path
        └── api.save_draft()
              POST /save-draft-data  → stored in DB, NOT live yet
              Server returns confirmationPrompt → waits for user to say "yes"
              User confirms → confirm_offer_update() called
              → api.promote_offer_draft() → PUT without saveType → backend reads draft from DB → goes live
```

---

### Production auth flow

```
User in Cursor → HTTP request to MCP server
  │
  ▼
CFAuthMiddleware (middleware.py)
  │
  ├── Extract JWT from Authorization: Bearer header
  │
  ├── auth.py: validate_cf_token()
  │     ├── Fetch Cloudflare JWKS (cached 1 hour)
  │     ├── Match JWT kid to public key
  │     └── Verify signature + expiry + audience (CF_AUD_TAG)
  │
  ├── Set current_identity contextvar (email, sub, groups)
  │
  ▼
Tool handler runs → _get_user_creds()
  │
  ├── Reads current_identity → has CF identity
  │
  ├── token_store.get(sub) → check cached Laravel Passport token
  │
  └── Cache miss → api.laravel_login() → POST /api/auth/login with CF JWT
        → Laravel verifies CF JWT, returns Passport access_token
        → Cached in token_store keyed by CF sub
        → Used as Authorization: Bearer for all subsequent API calls
```

---

## 5. Module Reference — Every File Explained

### `mcp_server/__main__.py` — Entry point

The file Python runs when you do `python3 -m mcp_server`. Decides which transport to start based on the `MCP_TRANSPORT` environment variable.

```python
transport = os.environ.get("MCP_TRANSPORT", "stdio").lower()

if transport == "sse":
    # Build Starlette HTTP app with CFAuthMiddleware, run via uvicorn
    uvicorn.run(app, host=host, port=port)
else:
    # Default: local stdio mode
    mcp.run(transport="stdio")
```

**Why all logging goes to stderr:**  
stdout is used for JSON-RPC messages. Any text written there would break JSON parsing. Logging is always routed to stderr so it never touches the transport.

---

### `mcp_server/server.py` — Tool definitions

Defines all MCP tools using `@mcp.tool()` decorator. The LLM reads each tool's `name` and `description` to decide when to call it.

**Tools registered:**

| Tool | Type | What it does |
|---|---|---|
| `login` | Auth | Exchange CF JWT for Laravel Passport token (lazy — auto-called by other tools) |
| `list_offers` | Read | List all offers; filter by `offer_type` or `sub_type` |
| `get_offer` | Read | Fetch full details of a single offer by UUID |
| `update_offer` | Write | Change fields; auto-routes to DIRECT or DRAFT based on safety rules |
| `confirm_offer_update` | Write | Promote a saved draft to live after user confirms |

**Why tools never import httpx directly:**  
All HTTP calls go through `api.py`. This keeps tools thin and testable — you can test the tool logic without needing a running Laravel server.

---

### `mcp_server/api.py` — HTTP client layer

All Laravel API calls live here. Every function:
- Takes `settings` as first argument (no module-level state)
- Takes optional `user_token` and `cf_jwt` for per-user auth
- Returns `{"success": True/False, ...}` dicts — never raises exceptions to the caller
- Logs every request and response (with sensitive headers truncated) at INFO level

**Endpoints used:**

```
GET  /api/admin/offer/get-offer/{uuid}                    → get_offer()
GET  /api/admin/offer/get-offer-list                      → list_offers()
POST /api/admin/offer/save-draft-data                     → save_draft()
PUT  /api/admin/offer/update-offer-complete/{type}/update → update_offer_direct()
                                                            promote_offer_draft()
```

**The `saveType` field:**

```
saveType='direct'  → backend uses request body immediately (no draft read)
saveType missing   → backend reads draft from DB (isPromoting = true)
```

---

### `mcp_server/config.py` — Settings

Loads all configuration from `.env` using `pydantic-settings`. The `.env` is always resolved relative to the project root regardless of where Python is launched from.

**Key settings:**

```
API_BASE_URL          → Laravel API URL (e.g. http://127.0.0.1:8000)
API_TOKEN             → Static Bearer token for local dev
AUTH_ENABLED          → false=stdio/local, true=HTTP/production
MCP_TRANSPORT         → stdio (default) or sse
CF_JWKS_URL           → Cloudflare JWKS endpoint (production only)
CF_AUD_TAG            → Cloudflare Access audience tag (production only)
MCP_PUBLIC_URL        → Public URL of this server (for OAuth discovery)
LOG_LEVEL             → INFO or DEBUG
```

---

### `mcp_server/middleware.py` — Auth middleware (production only)

`CFAuthMiddleware` is an ASGI middleware that runs in HTTP/SSE mode only. For every request:

1. Extracts JWT from `Authorization: Bearer` header (or `Cf-Access-Jwt-Assertion`)
2. Validates it via `auth.py`
3. Sets `current_identity` contextvar
4. Passes request to the MCP handler

Health check paths (`/health`, `/healthz`, `/ready`) bypass auth so K8s probes always work.

---

### `mcp_server/auth.py` — JWT validation

Validates Cloudflare Access JWTs:

1. Decodes JWT header to get `kid` (key ID)
2. Fetches Cloudflare's public JWKS (cached 1 hour in memory)
3. Matches `kid` to a public key
4. Verifies signature (RS256), expiry, and audience against `CF_AUD_TAG`
5. Returns a `CFIdentity(email, sub, groups)` dataclass

The `current_identity` contextvar is how identity flows from middleware → tool handlers without passing it through every function call.

---

### `mcp_server/offer_filter.py` — Safety routing logic

Python mirror of the Angular frontend's offer-step evaluation logic. Answers two questions:

**1. Should this change go DIRECT or DRAFT?** (`check_draft_or_live`)

```python
# Fields that always require draft (human review):
KEY_PREFIX_LIST_FOR_DRAFT_CHECKING = [
    "categoryId", "maxData.campaignData.budget", "is_ping",
    "prePing", "prePop", "offerTag", "offerBid", ...
]
# offerBid is special: only draft if |% change| >= 30%
```

**2. Which integration steps need to run?** (`evaluate_offer_steps`)

When saving a draft, the backend needs to know which downstream systems to resync (MAX campaigns, HasOffers, LEADGEN, etc.). This mirrors the Angular `OfferStepsService.evaluateOfferSteps` function exactly.

**Why mirror Angular logic here?**  
The `save-draft-data` API requires an `executeSteps` array. The Angular frontend computes this client-side. Without mirroring it here, the MCP server would have to depend on the frontend being available or the API would receive an incomplete payload.

---

### `mcp_server/token_store.py` — Token cache

In-memory store for Laravel Passport tokens, keyed by Cloudflare user `sub`. Avoids calling `/api/auth/login` on every single tool call.

```
First call for user → laravel_login() → Passport token cached with expiry
Subsequent calls → token returned from cache
401 response → token invalidated → next call re-logins automatically
```

---

### `mcp_server/llm.py` — LLM helper

Thin wrapper around the FastRouter OpenAI-compatible API. Used by LLM-powered tools (currently `generate_creative_html`, which is commented out). Not needed for data-passthrough tools like `list_offers`, `get_offer`.

---

### `pyproject.toml` — Package definition

Makes `mcp_server` installable as a Python package. This is critical: without this, running `python3 -m mcp_server` only works when Python's `cwd` is exactly the project root. With `pip install -e .`, Python always finds `mcp_server` regardless of where it's called from.

---

## 6. Local Development Setup

### Prerequisites

- Python 3.12
- Running local Laravel API (`php artisan serve` → `http://127.0.0.1:8000`)
- Cursor IDE

### Step 1 — Clone and install

```bash
cd /path/to/leadgen-dashboard-mcp
pip3 install -e .          # install as editable package (important!)
pip3 install -r requirements.txt
```

**Why `pip install -e .`?**  
This registers `mcp_server` as a known package in Python's site-packages — pointing at your local files. Cursor launches `python3 -m mcp_server` from its own process which may have a different working directory. Without this, Python can't find the package and you get `No module named mcp_server`.

### Step 2 — Create `.env`

```bash
cp .env.example .env
```

Edit `.env` — the minimum required for local dev:

```env
# Point at your local Laravel instance
API_BASE_URL=http://127.0.0.1:8000

# Static token from your local Laravel user (php artisan tinker → create token)
API_TOKEN=your-local-token-here

# Leave these empty for local dev:
AUTH_ENABLED=false
CF_JWKS_URL=
CF_AUD_TAG=
```

### Step 3 — Add to Cursor MCP config

Edit `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "ops-mcp-local": {
      "command": "/Library/Frameworks/Python.framework/Versions/3.12/bin/python3",
      "args": ["-m", "mcp_server"],
      "cwd": "/path/to/leadgen-dashboard-mcp"
    }
  }
}
```

### Step 4 — Enable in Cursor

**Settings → MCP** → toggle `ops-mcp-local` on. You should see these tools appear:

- `login`
- `list_offers`
- `get_offer`
- `update_offer`
- `confirm_offer_update`

### Step 5 — Verify

Test it by asking Cursor: _"list all offers"_ or _"get offer {uuid}"_

---

### What changes between local and production

| Setting | Local dev | Production |
|---|---|---|
| `MCP_TRANSPORT` | `stdio` (default) | `sse` |
| `AUTH_ENABLED` | `false` | `true` |
| `API_BASE_URL` | `http://127.0.0.1:8000` | `https://your-laravel-api.com` |
| `API_TOKEN` | your personal dev token | empty (uses CF JWT flow) |
| `CF_JWKS_URL` | empty | `https://your-team.cloudflareaccess.com/...` |
| `CF_AUD_TAG` | empty | your CF Access app audience tag |
| `MCP_PUBLIC_URL` | `http://localhost:8080` | `https://mcp.your-domain.com` |
| Cursor config | `command + args + cwd` | `url` only |
| Docker | not needed | `docker build && docker run` |

---

## 7. Debugging Lessons (What We Learned)

These are real issues we hit and solved — useful if you see them again.

---

### Problem: Cursor shows wrong/old tools after updating server.py

**Root cause:** Two things can cause this:

1. **Stale tool cache** — Cursor caches tool descriptors in:
   ```
   ~/.cursor/projects/<project-hash>/mcps/user-<server-name>/tools/
   ```
   These are JSON files Cursor writes when it first connects. They don't auto-update.

2. **Wrong package installed** — If an old `mcp_server` is installed via `pip install -e .` pointing at a different directory, Python loads that instead of your current code. Check with:
   ```bash
   pip3 show mcp-server
   # Look at "Editable project location" — must point to THIS project
   ```

**Fix:**
```bash
# 1. Uninstall wrong package
pip3 uninstall mcp-server -y

# 2. Reinstall from correct directory
cd /path/to/leadgen-dashboard-mcp
pip3 install -e .

# 3. Delete stale cache
rm ~/.cursor/projects/<hash>/mcps/user-ops-mcp-local/tools/*.json

# 4. Toggle server off/on in Cursor Settings → MCP
```

---

### Problem: `No module named mcp_server`

**Root cause:** `pip install -e .` was never run (or was uninstalled). Cursor launches the process from a different cwd than expected.

**Fix:**
```bash
cd /path/to/leadgen-dashboard-mcp
pip3 install -e .
```

---

### Problem: Closing and reopening the Cursor window doesn't reload tools

**Why:** Cursor is an Electron app. Closing a window doesn't kill the main process. The MCP server subprocess keeps running with the old code in memory.

**Fix:** `Cmd + Q` to fully quit Cursor, then reopen. Or kill the process:
```bash
pkill -f "python3 -m mcp_server"
```

---

### Problem: Multiple MCP servers with the same package name conflict

**Why:** If you have multiple entries in `mcp.json` pointing to different directories (old repo vs new repo), and both expose a package named `mcp_server`, Python resolves whichever is in site-packages first.

**Fix:** Make sure only ONE editable install exists, pointing to the correct project:
```bash
pip3 show mcp-server  # check the "Editable project location"
pip3 uninstall mcp-server -y
pip3 install -e /path/to/correct/project
```
