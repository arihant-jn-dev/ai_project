# Cloudflare Worker — Deployment Guide

> **One-stop reference for deploying, configuring, and updating the OAuth Worker.**

---

## Table of Contents

1. [What You Are Deploying](#1-what-you-are-deploying)
2. [How the Login Flow Works](#2-how-the-login-flow-works)
3. [Worker Routes — What Each Endpoint Does](#3-worker-routes--what-each-endpoint-does)
4. [Before You Start — What to Collect from Cloudflare](#4-before-you-start--what-to-collect-from-cloudflare)
5. [Step-by-Step: First Deployment](#5-step-by-step-first-deployment)
6. [What to Configure on Your MCP Server After Deploy](#6-what-to-configure-on-your-mcp-server-after-deploy)
7. [How to Re-deploy After a Code Change](#7-how-to-re-deploy-after-a-code-change)
8. [How to Update a Secret or Config Value](#8-how-to-update-a-secret-or-config-value)
9. [Full Map — What Comes From Where](#9-full-map--what-comes-from-where)
10. [Troubleshooting](#10-troubleshooting)
11. [What is Wrangler and Why Is It So Important?](#11-what-is-wrangler-and-why-is-it-so-important)

---

## 1. What You Are Deploying

This repo **is** the Cloudflare Worker. Commands run from the repo root.

```
dashboard-mcp-worker/       ← repo root, run all commands here
├── wrangler.toml           ← config (KV namespace ID already filled in)
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts            ← entry point + Env interface
    └── access-handler.ts   ← /authorize login logic
```

**What this Worker does:**
- Handles the OAuth login flow when Cursor/Claude connects to your MCP server for the first time
- In **production**: CF Access sits in front of the Worker and intercepts `/authorize` before the Worker code runs, forcing the user to log in and setting a `CF_Authorization` cookie. The Worker reads that cookie and completes the OAuth handshake.
- In **local dev**: No CF Access in front, so the Worker manually redirects to the CF Access login page.
- **After login it is completely idle** — all tool calls go directly to your MCP server, not through this Worker.

**What this Worker does NOT do:**
- It does NOT proxy tool calls
- It does NOT touch your MCP server during tool execution
- It does NOT act as an OIDC client — there is no client secret or code exchange

---

## 2. How the Login Flow Works

### Production (CF Access in front of Worker)

```
Cursor
  │
  │  1. POST http://mcp-stage.customer-acquisition.co/
  │     ← 401 + WWW-Authenticate: Bearer resource_metadata=".../.well-known"
  │
  │  2. GET /.well-known/oauth-authorization-server
  │     ← { "authorization_endpoint": "https://<worker>.workers.dev/authorize",
  │          "token_endpoint":         "https://<worker>.workers.dev/token",
  │          "registration_endpoint":  "https://<worker>.workers.dev/register" }
  │
  │  3. POST /register   → Worker returns { client_id }
  │
  │  4. Opens browser → /authorize
  │         ↓
  │   ┌─────────────────────────────────────────────────┐
  │   │  CF Access intercepts /authorize at the edge    │
  │   │  BEFORE Worker code runs.                       │
  │   │  User logs in (Google / email OTP).             │
  │   │  CF Access sets CF_Authorization cookie.        │
  │   │  CF Access redirects back to /authorize.        │
  │   └─────────────────────────────────────────────────┘
  │         ↓
  │   Worker runs — finds CF_Authorization cookie
  │   → calls completeAuthorization → redirects Cursor
  │     to its redirect_uri with the auth code
  │
  │  5. POST /token (Cursor sends auth code)
  │     ← { "access_token": "<CF JWT>" }
  │
  │  6. POST / (all future tool calls — direct to MCP server)
  │     Authorization: Bearer <CF JWT>
  │     ← 200 OK
```

### Key point

CF Access is **invisible to Cursor**. Cursor only knows the Worker URL. CF Access appears only as a login page in the user's browser at step 4.

---

## 3. Worker Routes — What Each Endpoint Does

Your Worker runs at: `https://leadgen-mcp-oauth.tech-2e7.workers.dev`

| Route | Method | Who calls it | What it does |
|-------|--------|-------------|-------------|
| `/register` | POST | Cursor / Claude | Dynamic client registration. Returns a `client_id` Cursor uses for the rest of the flow. **CF Access bypass policy required** (no login needed). |
| `/authorize` | GET | Browser (opened by Cursor) | CF Access intercepts this in production before Worker runs. After login, Worker reads the `CF_Authorization` cookie and completes OAuth authorization. |
| `/token` | POST | Cursor / Claude | Handled by OAuthProvider library. Returns the CF JWT as `access_token`. **CF Access bypass policy required** (no login needed). |

### Routes NOT handled by this Worker

| Route | Where it goes | What it does |
|-------|--------------|-------------|
| `POST /` | MCP server | Tool calls. Auth middleware validates the JWT. |
| `GET /.well-known/oauth-authorization-server` | MCP server | OAuth discovery. Points Cursor at the Worker URL. |

### How the routes flow together

```
Step 1:  Cursor → POST /register
         Worker → { client_id }

Step 2:  Cursor → opens GET /authorize in browser
         CF Access intercepts → shows login page
         User logs in → CF sets CF_Authorization cookie
         CF redirects back to /authorize
         Worker reads cookie → completeAuthorization
         Worker → 302 to Cursor's redirect_uri?code=abc

Step 3:  Cursor → POST /token?code=abc
         Worker → { access_token: "<CF JWT>" }

         ── Worker is DONE. Never called again. ──

Step 4+: Cursor → POST / (tool calls, direct to MCP server)
         MCP server validates JWT → 200 OK
```

---

## 4. Before You Start — What to Collect from Cloudflare

You need **2 things** before you can deploy.

---

### Thing 1 — KV Namespace ID

> **Already done** — the KV ID is already set in `wrangler.toml`:
> `id = "0f067e47f6274007922e8b1ec5cf3e2e"`
>
> Skip to Thing 2 unless you need to create a fresh namespace.

**If you ever need to create a new one:**

```
dash.cloudflare.com
  → Workers & Pages (left sidebar)
    → KV (tab at the top)
      → Create a namespace
        Name: OAUTH_KV → click Add

Copy the Namespace ID and paste it into wrangler.toml:
[[kv_namespaces]]
id = "<paste here>"
```

---

### Thing 2 — Cookie Encryption Key

**What it is:** A random 32-byte key used to encrypt OAuth session cookies. You generate it yourself — it never comes from Cloudflare.

**How to generate:**
```bash
openssl rand -hex 32
# Example: a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
```

**This goes into a Wrangler secret** (set during deployment in Step 5):
```bash
npx wrangler secret put COOKIE_ENCRYPTION_KEY
# paste the generated key when prompted
```

---

### Thing 3 — CF Access Self-Hosted Application (one-time dashboard setup)

In production, CF Access needs to sit **in front of** the Worker. You set this up as a **Self-Hosted** application in Zero Trust — not SaaS / OIDC.

```
dash.cloudflare.com → Zero Trust → Access → Applications
  → Add an application → Self-hosted

  Application name:  leadgen-mcp-oauth
  Application domain: leadgen-mcp-oauth.tech-2e7.workers.dev

  → Next → Add a policy:
    Policy name: Allow team
    Action: Allow
    Include: Emails ending in @customer-acquisition.co
    → Save

  → Next → Advanced settings:
    Cookie settings: HttpOnly ✓, SameSite: None
    → Save
```

**Then add bypass policies for `/register` and `/token`** — Cursor calls these in the background (no browser, no cookie):

```
Zero Trust → Access → Applications → leadgen-mcp-oauth → Policies
  → Add a policy:
    Policy name: Bypass /register
    Action: Bypass
    Path: /register
    → Save

  → Add a policy:
    Policy name: Bypass /token
    Action: Bypass
    Path: /token
    → Save
```

This ensures:
- `/authorize` → CF Access enforces login (sets `CF_Authorization` cookie)
- `/register` and `/token` → CF Access lets Cursor through without a browser login

---

## 5. Step-by-Step: First Deployment

Run all commands from the repo root (`dashboard-mcp-worker/`).

---

### Step 5.1 — Install dependencies

```bash
npm install
```

---

### Step 5.2 — Log in to Cloudflare

```bash
npx wrangler login
```

Your browser opens. Click **Allow**. Return to the terminal.

Verify:
```bash
npx wrangler whoami
# ✅  You are logged in with an API Token!
# Account Name: Customer Acquisition   Account ID: abc123...
```

---

### Step 5.3 — Check `wrangler.toml`

The file is already configured. Verify it looks like this:

```toml
name = "leadgen-mcp-oauth"
main = "src/index.ts"
compatibility_date = "2024-11-01"
compatibility_flags = ["nodejs_compat"]

[[kv_namespaces]]
binding = "OAUTH_KV"
id      = "0f067e47f6274007922e8b1ec5cf3e2e"

[vars]
CF_TEAM_DOMAIN = "custac.cloudflareaccess.com"
```

Nothing to change here unless you are using a different KV namespace.

---

### Step 5.4 — Set the only secret

```bash
# Generate a key and copy the output
openssl rand -hex 32

# Paste it when prompted
npx wrangler secret put COOKIE_ENCRYPTION_KEY
```

Verify it saved:
```bash
npx wrangler secret list
# NAME                    MODIFIED
# COOKIE_ENCRYPTION_KEY   2026-04-15T10:00:00Z
```

---

### Step 5.5 — Check TypeScript (optional but recommended)

```bash
npx tsc --noEmit
# No output = no errors ✅
```

---

### Step 5.6 — Deploy

```bash
npm run deploy
```

Successful output:
```
✅  Uploaded leadgen-mcp-oauth (2.34 sec)
✅  Deployed leadgen-mcp-oauth triggers (0.26 sec)
   https://leadgen-mcp-oauth.tech-2e7.workers.dev
```

The Worker is now live globally.

---

### Step 5.7 — Set up CF Access in front of the Worker (if not done yet)

Follow the instructions in [Thing 3 above](#thing-3--cf-access-self-hosted-application-one-time-dashboard-setup).

This only needs to be done once — CF Access configuration persists independently of Worker deployments.

---

## 6. What to Configure on Your MCP Server After Deploy

Open `leadgen-dashboard-mcp/.env` and set:

```env
# Worker URL — points Cursor at this Worker for OAuth
OAUTH_WORKER_URL=https://leadgen-mcp-oauth.tech-2e7.workers.dev

# MCP server public URL
MCP_PUBLIC_URL=https://mcp-stage.customer-acquisition.co

# CF Access JWT validation — always the team domain certs endpoint
CF_JWKS_URL=https://custac.cloudflareaccess.com/cdn-cgi/access/certs

# AUD tag from Zero Trust → Access → Applications → your app → Overview
CF_AUD_TAG=f5a60af8b11d737cd11e73067a3bf5c013929511bef5eefa1038fa4065ef440e

# Enable auth
AUTH_ENABLED=true
```

**Where each value comes from:**

| `.env` key | Where to get it |
|------------|----------------|
| `OAUTH_WORKER_URL` | Already known: `https://leadgen-mcp-oauth.tech-2e7.workers.dev` |
| `MCP_PUBLIC_URL` | Your K8s ingress / staging URL |
| `CF_JWKS_URL` | Always: `https://custac.cloudflareaccess.com/cdn-cgi/access/certs` |
| `CF_AUD_TAG` | Zero Trust → Access → Applications → your app → Overview → AUD tag |
| `AUTH_ENABLED` | Set to `true` for staging/prod |

---

## 7. How to Re-deploy After a Code Change

If you change anything in `src/index.ts` or `src/access-handler.ts`:

```bash
# Optional: check for TypeScript errors first
npx tsc --noEmit

# Deploy
npm run deploy
```

Wrangler detects the Worker already exists (by name in `wrangler.toml`) and updates it in place. **New code is live within seconds** — no dashboard changes needed.

---

## 8. How to Update a Secret or Config Value

### Update `CF_TEAM_DOMAIN` (plain var in `wrangler.toml`)

1. Edit `wrangler.toml` — change the value
2. Run `npm run deploy`

```bash
# Edit wrangler.toml, then:
npm run deploy
```

### Rotate `COOKIE_ENCRYPTION_KEY`

```bash
openssl rand -hex 32        # generate a new key, copy it
npx wrangler secret put COOKIE_ENCRYPTION_KEY   # paste it when prompted
```

No re-deploy needed — secret changes take effect immediately.

> **Note:** Rotating this key invalidates all in-flight OAuth sessions. Users mid-login will need to start over (they just click Sign in again).

---

## 9. Full Map — What Comes From Where

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  VALUE                    SOURCE                     GOES INTO               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  KV Namespace ID          CF Dashboard → Workers      wrangler.toml         │
│                           → KV → OAUTH_KV             (already set ✓)       │
│                                                                              │
│  COOKIE_ENCRYPTION_KEY    openssl rand -hex 32         wrangler secret put  │
│                           (generate yourself)          (never in files)     │
│                                                                              │
│  CF_TEAM_DOMAIN           Already set ✓                wrangler.toml [vars] │
│                           custac.cloudflareaccess.com                        │
│                                                                              │
│  OAUTH_WORKER_URL         Already known ✓              MCP server .env      │
│                           leadgen-mcp-oauth.tech-2e7.workers.dev            │
│                                                                              │
│  CF_JWKS_URL              Always the team domain       MCP server .env      │
│                           /cdn-cgi/access/certs        (already set ✓)      │
│                                                                              │
│  CF_AUD_TAG               Zero Trust → Access →        MCP server .env      │
│                           Applications → your app      (already set ✓)      │
│                           → Overview → AUD tag                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Troubleshooting

### Cursor gets a 401 loop — never sees a login page

The CF Access application is not set up in front of the Worker, or the bypass policies for `/register` and `/token` are missing. Check:

```
Zero Trust → Access → Applications
  → Is "leadgen-mcp-oauth" listed?
  → Does it cover leadgen-mcp-oauth.tech-2e7.workers.dev?
  → Does it have bypass policies for /register and /token?
```

---

### `/authorize` redirects to CF Access login but never comes back to Cursor

This is expected for local dev — CF Access won't redirect back to `localhost`. Use the deployed Worker URL for the full round-trip.

In production, this usually means the Access policy is blocking the return redirect. Check that the user's email is covered by the Allow policy.

---

### "KV namespace not found" error on deploy

```
Error: KV namespace with ID "..." not found
```

The `id` in `wrangler.toml` doesn't match a real namespace in your Cloudflare account. Create the namespace in the dashboard and paste the correct ID.

---

### Worker deployed but MCP server points at the wrong Worker URL

Check `OAUTH_WORKER_URL` in your MCP server `.env`:

```env
OAUTH_WORKER_URL=https://leadgen-mcp-oauth.tech-2e7.workers.dev
```

Restart the MCP server after changing `.env`.

---

### `COOKIE_ENCRYPTION_KEY` not set — Worker returns 500

```bash
npx wrangler secret list
# If COOKIE_ENCRYPTION_KEY is missing:
openssl rand -hex 32
npx wrangler secret put COOKIE_ENCRYPTION_KEY
```

---

### Users get "Access Denied" after logging in

The CF Access Allow policy doesn't cover their email. Check:

```
Zero Trust → Access → Applications → leadgen-mcp-oauth → Policies
  → Allow team policy → Include rule → Emails ending in @customer-acquisition.co
```

---

## 11. What is Wrangler and Why Is It So Important?

### What Wrangler is

Wrangler is the **official CLI tool made by Cloudflare** for building, testing, and deploying Cloudflare Workers. It is the only official way to push Worker code to Cloudflare from the terminal.

| Tool | What it does |
|------|-------------|
| `git` | Pushes code to GitLab/GitHub |
| `kubectl` | Deploys apps to Kubernetes |
| `wrangler` | Deploys Worker code to Cloudflare |

---

### What Wrangler does in this project

#### 1. Authenticates you with Cloudflare

```bash
npx wrangler login
```

Opens your browser, you log in with your Cloudflare account, and Wrangler stores a token at `~/.wrangler/config/default.toml`. Every command after this uses that token.

---

#### 2. Compiles and deploys your TypeScript code

```bash
npm run deploy   # runs: wrangler deploy
```

Wrangler does all of this in one command:

```
Step 1: Reads wrangler.toml
          - Worker name (leadgen-mcp-oauth)
          - Entry point (src/index.ts)
          - KV binding (OAUTH_KV)
          - Env vars (CF_TEAM_DOMAIN)

Step 2: Compiles TypeScript → JavaScript bundle

Step 3: Bundles all dependencies into a single file
        (no node_modules shipped — one optimised bundle)

Step 4: Uploads to Cloudflare's edge network
        (deploys globally to 300+ data centres instantly)

Step 5: Prints the Worker URL
        https://leadgen-mcp-oauth.tech-2e7.workers.dev
```

---

#### 3. Manages secrets safely

```bash
npx wrangler secret put COOKIE_ENCRYPTION_KEY
```

Secrets are sent directly to Cloudflare's encrypted vault over HTTPS. They:
- Never touch your filesystem
- Never appear in `wrangler.toml` (which is committed to git)
- Never appear in CI logs
- Are injected into your Worker at runtime as `env.COOKIE_ENCRYPTION_KEY`

---

#### 4. Lets you test locally before deploying

```bash
npm run dev   # runs: wrangler dev --port 8787
```

Starts a local Worker runtime at `http://localhost:8787`. Combine with the MCP server running locally to test the full OAuth chain before touching production.

See `local_setup.md` for the complete local testing guide.

---

#### 5. Streams live logs from your deployed Worker

```bash
npx wrangler tail
```

Shows real-time logs as requests come in:

```
[2026-04-15 12:00:01] GET /authorize - 302 Redirect to CF Access login
[2026-04-15 12:00:05] GET /authorize - 302 Redirect to Cursor callback (cookie found)
[2026-04-15 12:00:05] POST /token - 200 OK
```

---

### Why `npx wrangler` instead of just `wrangler`

`npx` runs Wrangler from your local `node_modules/` (installed by `npm install`). This ensures everyone on the team uses the exact same version defined in `package.json`, not a random global install.

```json
"devDependencies": {
  "wrangler": "^4.83.0"    ← this version is always used
}
```

---

### Wrangler vs Cloudflare Dashboard

| Task | Wrangler CLI | Cloudflare Dashboard |
|------|-------------|---------------------|
| Deploy code | `npm run deploy` ✅ | Manual copy-paste ❌ |
| Set secrets | `wrangler secret put` ✅ | Can also do it here |
| Create KV | `wrangler kv namespace create` ✅ | Can also do it here |
| View logs | `wrangler tail` ✅ | Limited log viewer |
| CI/CD pipeline | Works in GitLab CI ✅ | Not possible ❌ |
| CF Access policies | Not possible ❌ | Must do in dashboard ✅ |
| Bypass policies | Not possible ❌ | Must do in dashboard ✅ |

**Rule of thumb:**
- Code, secrets, KV → use Wrangler
- CF Access policies, Zero Trust rules → use the dashboard

---

### One-line summary

> **Wrangler = the bridge between your code on your laptop and the live Worker running on Cloudflare's global network.**

---

## Quick Reference — Commands

```bash
# Run all commands from the repo root (dashboard-mcp-worker/)

# First time setup
npm install
npx wrangler login
openssl rand -hex 32                          # generate key, copy it
npx wrangler secret put COOKIE_ENCRYPTION_KEY # paste it
npm run deploy

# Re-deploy after code changes
npx tsc --noEmit    # check for TypeScript errors (optional)
npm run deploy

# Rotate the cookie key
openssl rand -hex 32
npx wrangler secret put COOKIE_ENCRYPTION_KEY

# Check what secrets are set
npx wrangler secret list

# Test locally (see local_setup.md for full setup)
npm run dev

# View live Worker logs
npx wrangler tail
```

---

## 🪵 Testing Live Logs on Deployed Worker

Once your worker is deployed, use the commands below to stream real-time logs while
triggering the OAuth flow from Cursor/Claude or a browser.

### Stream all live logs (pretty printed)
```bash
npx wrangler tail --format pretty
```

### Stream logs — errors only
```bash
npx wrangler tail --format pretty --status error
```

### Stream logs — successful requests only
```bash
npx wrangler tail --format pretty --status ok
```

### Stream logs as raw JSON (useful for piping / grepping)
```bash
npx wrangler tail --format json
```

### Filter logs to a specific endpoint path
```bash
# Only show /token requests
npx wrangler tail --format pretty | grep '"path":"/token"'

# Only show /authorize requests
npx wrangler tail --format pretty | grep '"path":"/authorize"'

# Only show /register requests
npx wrangler tail --format pretty | grep '"path":"/register"'
```

### What to look for per endpoint

**`POST /register`** — Cursor/Claude registering itself as an OAuth client.
```
event: "request"  label: "POST /register"
event: "response" label: "POST /register"  status: 200
```

**`GET /authorize`** — Browser hitting the auth page after Cursor/Claude initiates login.

No CF cookie yet (first visit — redirecting to CF Access login):
```
event: "authorize"  stage: "no_cf_cookie"  action: "redirect_to_cf_access_login"
```

CF cookie present (user already logged in — completing OAuth):
```
event: "authorize"  stage: "cf_cookie_found"  action: "completing_oauth"  user: "you@company.com"
event: "authorize"  stage: "auth_code_stored"  user: "you@company.com"  hasCode: true
```

**`POST /token`** — Cursor/Claude exchanging the auth code for the CF JWT.

Success:
```
event: "request"   label: "POST /token"
event: "response"  label: "POST /token"  status: 200  tokenType: "Bearer"  expiresIn: 3600
```

Failure — code missing from request:
```
event: "response"  label: "POST /token"  status: 400  reason: "missing code param"
```

Failure — code expired or not found in KV (codes expire after 120 seconds):
```
event: "response"  label: "POST /token"  status: 404  reason: "no JWT found for code"
```

### Cloudflare Dashboard (no terminal needed)

1. Go to [dash.cloudflare.com](https://dash.cloudflare.com)
2. **Workers & Pages** → **`leadgen-mcp-oauth`** → **Logs** tab
3. Click **Begin log stream**
4. Trigger the flow from Cursor/Claude and watch logs appear in real time
```
