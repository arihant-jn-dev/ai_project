# Leadgen Ops AI Interface — Context Document

> **Read this file first** when returning to this project.  
> It explains every design decision, every file, how to run all modes,
> and how to extend the system.

---

## What Is This?

A **Model Context Protocol (MCP) server** that gives any AI assistant
(Cursor IDE or a terminal agent) the ability to query and update
advertising offers stored in the Leadgen dashboard-api backend — using
plain English.

Instead of writing curl commands or copy-pasting UUIDs into a dashboard:

```
"list all hostpost offers"
"what is the bid and budget for offer 2bd5b9b9-..."
"update the bid to $3.50 on offer 2bd5b9b9-..."
```

The AI picks the right tool, calls the real Laravel API, and summarises
only the fields relevant to the user's question.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  YOU                                                                 │
│  (Cursor chat  OR  python client.py in terminal)                     │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
           ┌────────────────┴─────────────────┐
           │                                  │
    MODE A: Cursor IDE                 MODE B: client.py
    Cursor's built-in LLM              FastRouter LLM
    (Claude, GPT-4, etc.)              (OpenAI-compatible proxy)
           │                                  │
           │  MCP stdio (JSON-RPC)            │  MCP stdio (JSON-RPC)
           │  Cursor spawns the server        │  client.py spawns the server
           │                                  │
           └────────────────┬─────────────────┘
                            │
             ┌──────────────▼──────────────┐
             │       mcp_server/           │
             │       server.py             │
             │   3 @mcp.tool() functions   │
             └──────────────┬──────────────┘
                            │  httpx  (Bearer auth + CF Access headers)
                            ▼
             ┌──────────────────────────────┐
             │  dashboard-api (Laravel)     │
             │  /api/admin/offer/*          │
             └──────────────────────────────┘
```

**Key insight:** `server.py` is always the same process doing the same work.
The only difference between modes is *who* spawns it and *which LLM* decides
which tools to call.

**Multi-user deployment:** The server also supports SSE (HTTP) transport.
Deploy it once; all team members connect via a URL instead of running a
local process. See `DEPLOYMENT.md` for details.

---

## File Map

```
ops_ai_interface/               ← project root (run all commands from here)
│
├── mcp_server/                 ← Python package (the MCP server)
│   ├── __init__.py             exports `mcp` instance
│   ├── __main__.py             `python -m mcp_server` entry point (stdio or SSE)
│   ├── config.py               Settings dataclass, reads .env
│   ├── api.py                  ALL httpx calls to Laravel API + auth logic
│   ├── offer_filter.py         Python mirror of Angular step-eval logic
│   └── server.py               FastMCP server + @mcp.tool() definitions
│
├── client.py                   Standalone CLI agent (Mode B)
├── requirements.txt            pip dependencies — install with pip install -r requirements.txt
├── .env.example                Template — copy to .env and fill in values
├── .env                        Your secrets (gitignored — never commit)
├── .gitignore
├── Dockerfile                  Builds the server image for shared SSE deployment
├── docker-compose.yml          Easiest way to run the shared server
├── README.md                   Setup and usage guide
├── DEPLOYMENT.md               Guide for going from local to shared team deployment
├── .cursor/
│   └── mcp.json                Cursor MCP configuration (gitignored — has secrets)
└── CONTEXT.md                  ← you are here
```

---

## File-by-File Details

### `mcp_server/config.py`

Reads configuration from environment variables / `.env` using
`pydantic-settings`. Exposes a module-level singleton `settings` that
every other file imports.

The `.env` path is resolved relative to `config.py` itself (not CWD),
so the server can be started from any directory.

| Variable                    | Default                              | Purpose                                                        |
| --------------------------- | ------------------------------------ | -------------------------------------------------------------- |
| `API_BASE_URL`              | `http://127.0.0.1:8000`              | Laravel API base URL (no trailing `/api`)                      |
| `API_TOKEN`                 | *(empty)*                            | Static Bearer token — if set, login flow is skipped entirely   |
| `DASHBOARD_EMAIL`           | *(empty)*                            | Login email for dynamic token acquisition (staging/prod)       |
| `DASHBOARD_PASSWORD`        | *(empty)*                            | Login password for dynamic token acquisition                   |
| `CF_ACCESS_CLIENT_ID`       | *(empty)*                            | Cloudflare Access service token ID (required when behind CF)   |
| `CF_ACCESS_CLIENT_SECRET`   | *(empty)*                            | Cloudflare Access service token secret                         |
| `FASTROUTER_BASE_URL`       | `https://go.fastrouter.ai/api/v1`    | FastRouter proxy URL                                           |
| `FASTROUTER_API_KEY`        | *(empty)*                            | FastRouter API key — required for Mode B only                  |
| `LLM_MODEL`                 | `anthropic/claude-sonnet-4-20250514` | Model to use via FastRouter                                    |
| `SERVER_NAME`               | `leadgen-ops`                        | MCP server display name in Cursor                              |
| `LOG_LEVEL`                 | `INFO`                               | Set to `DEBUG` to log all HTTP traffic to stderr               |
| `HTTP_DEBUG_MAX_BODY_CHARS` | `12000`                              | Truncation limit for debug logs                                |

**Transport env vars** (read by `__main__.py`, not `config.py`):

| Variable        | Default     | Purpose                                          |
| --------------- | ----------- | ------------------------------------------------ |
| `MCP_TRANSPORT` | `stdio`     | `stdio` for local Cursor use, `sse` for Docker   |
| `MCP_HOST`      | `0.0.0.0`   | Bind address (SSE mode only)                     |
| `MCP_PORT`      | `8080`      | Listen port (SSE mode only)                      |

---

### `mcp_server/api.py`

All HTTP calls to the Laravel API live here. Also owns the full auth
logic — `server.py` never imports `httpx` directly.

**Auth modes (evaluated in priority order):**

1. **Static token** — `API_TOKEN` is set → used as-is, no login call made
2. **Login flow** — `DASHBOARD_EMAIL` + `DASHBOARD_PASSWORD` set → server calls
   `POST /api/core/login`, caches the token for 1 hour, auto-refreshes on
   expiry or 401 (one retry per request)
3. **No auth** — all fields empty → requests sent without `Authorization`
   header (local dev only, when middleware is disabled)

**CF Access** headers (`CF-Access-Client-Id` / `CF-Access-Client-Secret`) are
added automatically to every request whenever `CF_ACCESS_CLIENT_ID` is set,
regardless of which auth mode is active.

| Function                     | HTTP Call                               | Purpose                  |
| ---------------------------- | --------------------------------------- | ------------------------ |
| `get_offer(settings, uuid)`  | `GET /api/admin/offer/get-offer/{uuid}` | Fetch single offer       |
| `list_offers(settings, ...)` | `GET /api/offer/get-offer-list`         | List offers with filters |
| `save_draft(settings, body)` | `POST /api/admin/offer/save-draft-data` | Save draft change        |

Internal helpers (not called by `server.py` directly):

| Helper              | Purpose                                                              |
| ------------------- | -------------------------------------------------------------------- |
| `_acquire_token`    | Calls login endpoint, caches result, returns token string            |
| `_invalidate_token` | Removes cached token so next call triggers a fresh login             |
| `_bearer_token`     | Resolves which token to use (static > login flow > none)             |
| `_auth_headers`     | Builds full header dict (Accept + CF headers + Authorization)        |
| `_request`          | Executes HTTP request; retries once with fresh token on 401          |
| `_cf_headers`       | Returns CF-Access headers when credentials are configured            |

---

### `mcp_server/offer_filter.py`

Python mirror of the Angular frontend's offer step-evaluation logic.

**Why this exists:**
When saving a draft via `save-draft-data`, the API requires an `executeSteps`
array that tells the backend which integration systems to re-sync
(MAX campaigns, HasOffers, LEADGEN, etc.). The Angular frontend computes
this array client-side. This file replicates that logic in Python so the
`update_offer_bid_draft` tool can build the correct payload without the
frontend being present.

**Mirrors these Angular files:**

- `offer.constants.ts` → `OFFER_FIELDS` (field → steps mapping)
- `offer-step-eval.service.ts` → `evaluate_offer_steps()`
- `offer-create-linkout-new.component.ts` → `check_draft_or_live()`

**Key functions:**

| Function                                                 | Purpose                                                                 |
| -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `steps_valid_for_offer_type(offer_type)`                 | Returns the ordered step list for hostpost/linkout/offerwall            |
| `evaluate_offer_steps(offer, activity_log, valid_steps)` | Computes which steps to run based on changed fields                     |
| `check_draft_or_live(activity_log)`                      | Returns True if changes are large enough to require draft/maker-checker |
| `build_live_offer_for_draft(live)`                       | Deep-copies live offer, strips server-only `offerActivityLog` key       |

**Step lists by offer type:**

| Offer Type                   | Steps                                                                                                                 |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `hostpost`                   | LEADGEN, MAX_CAMPAIGN, MAX_CAMPAIGN_TARGETTING, MAX_ADGROUP, MAX_ADGROUP_TARGETTING, MAX_ADGROUP_FREQUENCYCAP, MAX_AD |
| `linkout` / `custom-linkout` | Above + HASOFFERS_CREATE, HASOFFERS_SET_AFFILIATE, HASOFFERS_GEOTARGETING, HASOFFERS_TEST_LINK                        |
| `offerwall`                  | Same as linkout but WITHOUT HASOFFERS_GEOTARGETING                                                                    |

---

### `mcp_server/server.py`

The core FastMCP server. Defines all 3 tools with `@mcp.tool()`.

All tools are async functions. The LLM reads each tool's `name` and
`description` to decide when and how to call it.

**Design principle — one tool, one API call:**
Rather than having separate tools per data slice (`get_offer_bid`,
`get_offer_status`, `get_offer_max_data`, etc.) each making its own API
call, a single `get_offer` tool fetches the full offer object once. The
LLM surfaces only what the user asked for. This cuts redundant API calls
and removes the need for hardcoded field-mapping logic on the server side.

**Read tools:**

| Tool          | When the LLM uses it                                              |
| ------------- | ----------------------------------------------------------------- |
| `list_offers` | "list / show all offers", optionally filtered by type             |
| `get_offer`   | ANY question about a specific offer — bid, budget, status, config,|
|               | MAX campaign data, integration URLs, targeting, activity log, etc.|

**Write tools:**

| Tool                     | When the LLM uses it                                                    |
| ------------------------ | ----------------------------------------------------------------------- |
| `update_offer_bid_draft` | "update the bid to X on offer XYZ" — saves draft, does NOT push to live |

---

### `mcp_server/__main__.py`

Entry point for `python -m mcp_server`. Supports two transports selected
via the `MCP_TRANSPORT` environment variable:

| `MCP_TRANSPORT` | Transport | Use case                                                          |
| --------------- | --------- | ----------------------------------------------------------------- |
| `stdio` (default) | stdio   | Local dev — Cursor spawns this as a subprocess                    |
| `sse`           | HTTP/SSE  | Shared deployment — deployed once, all users connect via URL      |

In SSE mode, `MCP_HOST` (default `0.0.0.0`) and `MCP_PORT` (default `8080`)
control the bind address and port.

---

### `client.py`

Standalone CLI agent for Mode B. No Cursor required.

**What it does:**

1. Spawns `python -m mcp_server` as a subprocess (same as Cursor does)
2. Establishes an MCP session over stdio
3. Discovers all 3 tools from the server
4. Converts tool schemas to OpenAI function-calling format
5. Sends user queries to FastRouter with the tool list
6. Executes tool calls on the live MCP session
7. Feeds results back to the LLM until it has a complete answer
8. Prints the final answer

**Deduplication:** Some LLMs emit identical parallel tool calls in one
response. The agent caches by `(tool_name, args_json)` and reuses results.

---

## Running Modes

### Mode A — Cursor IDE (local stdio)

Cursor spawns `server.py` automatically. You type in Cursor's chat panel;
Cursor's model decides which tools to call.

**FastRouter is NOT needed for this mode.**

Best for: day-to-day ops work inside the IDE.

### Mode B — CLI Agent (`python client.py`)

You run `client.py` in a terminal. It spawns `server.py`, connects as
an MCP client, and uses FastRouter as the LLM brain.

**FastRouter IS required for this mode.**

Best for: testing tools, running without Cursor, automation scripts.

### Mode C — Shared SSE Server (team deployment)

The server runs once on a shared machine (VM / EC2 / Docker). All team
members connect via a URL in their `.cursor/mcp.json` — no local Python
setup needed on each machine.

See `DEPLOYMENT.md` for full setup instructions.

|                            | Mode A — Cursor                           | Mode B — CLI agent     | Mode C — Shared SSE    |
| -------------------------- | ----------------------------------------- | ---------------------- | ---------------------- |
| **Interface**              | Cursor chat                               | Terminal               | Cursor chat (remote)   |
| **LLM that routes tools**  | Cursor's model                            | FastRouter             | Cursor's model         |
| **FastRouter key needed?** | No                                        | Yes                    | No                     |
| **Python setup per user?** | Yes                                       | Yes                    | No                     |
| **Entry point**            | `python -m mcp_server` (Cursor spawns it) | `python client.py`     | `docker compose up`    |
| **mcp.json style**         | `command` + `args` + `cwd`                | N/A                    | `url: http://host/sse` |

---

## Available Tools

| Tool                     | What it does                                    |
| ------------------------ | ----------------------------------------------- |
| `list_offers`            | List offers with optional type/subType filters   |
| `get_offer`              | Full offer object — covers ALL data questions    |
| `update_offer_bid_draft` | Change bid → save as draft (never pushes live)   |

FastRouter is only used for the **agentic routing loop** in `client.py`.
The tools themselves always talk directly to the Laravel API over HTTP.

---

## Draft Save Flow (`update_offer_bid_draft`)

This tool replicates the exact sequence the Angular dashboard performs
when a user edits a bid and clicks "Save as Draft":

```
1. GET /api/admin/offer/get-offer/{uuid}
        │
        ▼
2. Extract response.live — the current live offer object
        │
        ▼
3. Deep-copy the live object (build_live_offer_for_draft)
   - Remove offerActivityLog (server-side history; draft uses activityLog)
        │
        ▼
4. Apply the new bid:  data["offerBid"] = new_offer_bid
        │
        ▼
5. Build activityLog:
   [{ fieldName: "offerBid", previousValue: old, currentValue: new }]
        │
        ▼
6. Run evaluate_offer_steps (offer_filter.py):
   - Determine which integration steps are affected by the bid change
   - For offerBid: LEADGEN, MAX_CAMPAIGN, MAX_ADGROUP,
                   MAX_ADGROUP_TARGETTING, HASOFFERS_CREATE
   - Filter to steps valid for the offer's type
   Also run check_draft_or_live → flags if |% change| >= 30
        │
        ▼
7. POST /api/admin/offer/save-draft-data
   Body: { "data": { ...live offer, offerBid: new, activityLog, executeSteps } }
        │
        ▼
8. Return summary to the LLM
   { success, previousOfferBid, newOfferBid, executeSteps, requiresDraft }
```

The draft is now visible in the dashboard. A human must promote it to live —
this tool deliberately does NOT do that.

---

## Setup & Running

### Step 1 — Python environment (one-time)

```bash
cd /path/to/ops_ai_interface

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Step 2 — Configure `.env`

```bash
cp .env.example .env
```

**Local dev** (auth middleware disabled):
```env
API_BASE_URL=http://127.0.0.1:8000
# Leave all auth fields empty
```

**Staging / Production** (Cloudflare Access + login flow):
```env
API_BASE_URL=https://your-dashboard.com
DASHBOARD_EMAIL=ops@company.com
DASHBOARD_PASSWORD=your-password
CF_ACCESS_CLIENT_ID=your-cf-client-id
CF_ACCESS_CLIENT_SECRET=your-cf-secret
```

For Mode B (CLI agent), also set:
```env
FASTROUTER_API_KEY=sk-v1-your-key-here
```

### Step 3 — Start Laravel API (local dev only)

```bash
php artisan serve   # → http://127.0.0.1:8000
```

### Mode A — Connect to Cursor

Create `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "leadgen-ops": {
      "command": "/path/to/ops_ai_interface/.venv/bin/python",
      "args": ["-m", "mcp_server"],
      "cwd": "/path/to/ops_ai_interface"
    }
  }
}
```

Go to **Cursor → Settings → MCP → Reload**. The `leadgen-ops` server
should appear as Connected with 3 tools.

> `.cursor/mcp.json` is gitignored — never commit it (it can contain secrets
> in the `env` block). Each developer creates their own copy.

### Mode B — CLI agent

```bash
source .venv/bin/activate
python client.py
```

### Mode C — Shared SSE server (Docker)

```bash
cp .env.example .env   # fill in staging/prod credentials
docker compose up -d
```

Each team member's `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "leadgen-ops": {
      "url": "http://<SERVER_IP>:8080/sse"
    }
  }
}
```

---

## Debugging

### Enable full HTTP logging

Set `LOG_LEVEL=DEBUG` in `.env`. All debug output goes to **stderr**
so it does not pollute the MCP stdio transport (which uses stdout).

### Common issues

| Symptom                          | Likely cause                              | Fix                                        |
| -------------------------------- | ----------------------------------------- | ------------------------------------------ |
| `API error: Unauthenticated`     | Auth credentials missing or wrong         | Check `.env` auth mode settings            |
| `401` on every request           | Expired token / wrong CF credentials      | Check CF_ACCESS_* and DASHBOARD_* in `.env`|
| Tools not appearing in Cursor    | Wrong path in `.cursor/mcp.json`          | Use absolute paths; use venv Python        |
| Cursor shows old tool list       | Cursor cached the old connection          | Settings → MCP → reconnect/reload          |
| `client.py` exits immediately    | `FASTROUTER_API_KEY` not set              | Add to `.env`                              |
| Draft saved but bid unchanged    | `new_offer_bid` type mismatch             | Ensure numeric, not string                 |
| DEBUG logs not appearing         | `LOG_LEVEL` not set to `DEBUG`            | Set in `.env` and restart                  |
| MCP connection drops             | Server crashed on startup                 | Check stderr for Python import errors      |
| SSE server not reachable         | Firewall blocking port / wrong IP         | Check Docker port mapping and firewall     |

---

## Adding a New Tool

**1. `api.py`** — add the HTTP call function

```python
def my_new_action(settings: Settings, offer_unique_id: str, ...) -> dict[str, Any]:
    url = f"{settings.api_base_url}/api/admin/offer/my-endpoint/{offer_unique_id}"
    try:
        resp = _request(settings, "GET", url)
        resp.raise_for_status()
        return resp.json()
    except httpx.HTTPStatusError as exc:
        return _handle_http_error(exc)
    except Exception as exc:
        return {"success": False, "error": str(exc)}
```

**2. `server.py`** — add the `@mcp.tool()` function

```python
@mcp.tool(
    name="my_new_tool",
    description="What this tool does — the LLM reads this to decide when to call it.",
)
async def my_new_tool(offer_unique_id: str, ...) -> dict[str, Any]:
    return api.my_new_action(settings, offer_unique_id, ...)
```

**3. Test without restarting Cursor:**

```bash
python client.py
# > use my new tool on offer XYZ
```

**4. Restart Cursor MCP** when satisfied (Settings → MCP → Reload).

### Suggested next tools (Phase 2)

| Tool                    | Endpoint                                             |
| ----------------------- | ---------------------------------------------------- |
| `update_offer_status`   | `POST /api/admin/offer/update-offer-status`          |
| `clone_offer`           | `POST /api/admin/offer/clone-offer/{uuid}`           |
| `get_activity_log`      | Included in `get-offer` response under `activityLog` |
| `list_hostpost_offers`  | `GET /api/admin/offer/get-offer-list-hostpost`       |
| `promote_draft_to_live` | (requires separate endpoint research)                |

---

## Postman Reference

Collection: `21149051-e671be07-16d8-48ac-84a5-093a419fdeb7`

| Request             | Method | URL                                 |
| ------------------- | ------ | ----------------------------------- |
| Ops get-offer       | GET    | `/api/admin/offer/get-offer/{uuid}` |
| Ops save-draft-data | POST   | `/api/admin/offer/save-draft-data`  |
| Ops get-offer-list  | GET    | `/api/offer/get-offer-list`         |

---

## Why FastRouter CLI?

| Situation                                        | Use CLI agent?                          |
| ------------------------------------------------ | --------------------------------------- |
| Testing a new tool just added to `server.py`     | Yes — faster than restarting Cursor     |
| Running offer ops from a terminal on a server    | Yes — no IDE needed                     |
| Sharing with a teammate who doesn't use Cursor   | Yes — works with just Python + `.env`   |
| Automating a repetitive update in a shell script | Yes — `--once` flag makes it scriptable |
| CI/CD pipeline that reads/updates offers         | Yes — headless, no GUI dependency       |
| Normal day-to-day work inside the IDE            | No — Cursor + MCP is simpler            |

```bash
# Batch script example: update bid on multiple offers
for uuid in uuid1 uuid2 uuid3; do
  python client.py --once "update bid to 3.50 on offer $uuid"
done
```
