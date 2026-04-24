# Ops MCP Server — Teammate Setup Guide

How to connect to the MCP server on **Cursor** or **Claude Desktop**.

---

## Prerequisites checklist

Before touching any config, make sure all of these are done:

| # | Requirement | Who sets it up |
|---|---|---|
| 1 | **Twingate access** — you must be added to the Twingate resource | Ask your admin |
| 2 | **Cloudflare Access** — your `@customer-acquisition.co` Google account must be allowed | Ask your admin |
| 3 | **Twingate client running** — the desktop app must be open and connected | You |
| 4 | **Node.js installed** *(Claude Desktop only)* | You (one-time) |

---

## Step 1 — Verify Twingate access

Open a terminal and run:

```bash
# Mac / Linux
curl http://dashboard-mcp-stage-service.dashboard-api-staging/

# Windows (CMD or PowerShell)
curl http://dashboard-mcp-stage-service.dashboard-api-staging/
```

**Expected:** any HTTP response (even a 401 or JSON error is fine — it means the server is reachable).

**If it hangs or says "could not resolve host":** Twingate is not connected or you have not been granted access to this resource. Contact your admin.

Do not proceed until this curl works.

---

## Step 2 — Connect in Cursor (no Node.js needed)

Cursor has built-in remote MCP support — just paste the URL.

Open `~/.cursor/mcp.json` (Mac/Linux) or `%APPDATA%\Roaming\Cursor\User\globalStorage\cursor.mcp\mcp.json` (Windows) and add:

```json
{
  "mcpServers": {
    "ops-mcp-stage": {
      "url": "http://dashboard-mcp-stage-service.dashboard-api-staging/"
    }
  }
}
```

Then:
1. Open Cursor → **Settings → MCP**
2. You should see `ops-mcp-stage` with a green dot and the tools listed
3. If it shows an error, check that Twingate is running

On first connect, Cursor will open a browser tab for Google login — sign in with your `@customer-acquisition.co` account.

---

## Step 3 — Connect in Claude Desktop (Node.js required)

Claude Desktop does not have built-in remote MCP support, so it needs a bridge called `mcp-remote` that runs via Node.js.

### What is mcp-remote and why is it needed?

MCP servers can be either **local** (a process running on your own machine) or **remote** (a server running somewhere on a network, like ours in Kubernetes).

Claude Desktop was originally designed only for **local** MCP servers — it spawns a process on your machine and talks to it directly. It has no native way to connect to a remote HTTP MCP server over a URL.

`mcp-remote` is a small Node.js bridge that fills this gap:

```
Claude Desktop
    │
    │  spawns a local process (like any local MCP server)
    ▼
mcp-remote (running on your machine)
    │
    │  translates Claude's local requests → HTTP calls over the network
    ▼
Our MCP Server (Kubernetes, behind Twingate)
```

It also handles the **OAuth login flow** — when Claude first connects, `mcp-remote` opens a browser tab for the Cloudflare/Google login and stores the token for subsequent calls.

**Why Cursor doesn't need it:** Cursor has this bridge built directly into the app. You just give it a URL and it handles everything internally. Claude Desktop hasn't added this yet, so `mcp-remote` is the workaround until they do.

### 3a — Install Node.js (one time only)

Download and install the **LTS version** from [https://nodejs.org](https://nodejs.org).

During installation, make sure **"Add to PATH"** is checked (it is by default).

After installing, open a **new** terminal and verify:

```cmd
node --version   # should print v18 or higher
npx --version    # should print a version number
```

If `npx --version` gives no output, see [Troubleshooting](#troubleshooting) below.

### 3b — Add config to Claude Desktop

Open Claude Desktop → **Settings → Developer → Edit Config**.

The config file is at `%APPDATA%\Claude\claude_desktop_config.json` on Windows or `~/Library/Application Support/Claude/claude_desktop_config.json` on Mac.

Add the `mcpServers` block:

```json
{
  "mcpServers": {
    "ops-mcp-stage-claude": {
      "command": "npx.cmd",
      "args": [
        "-y",
        "mcp-remote",
        "http://dashboard-mcp-stage-service.dashboard-api-staging/",
        "--allow-http"
      ]
    }
  }
}
```

> **Mac users:** use `"command": "npx"` instead of `"npx.cmd"`

### 3c — Fully restart Claude Desktop

Closing the window is not enough — the old process keeps running.

- **Windows:** right-click the Claude icon in the **system tray** (bottom-right corner) → **Quit**, then reopen
- **Mac:** `Cmd + Q`, then reopen

### 3d — Login on first connect

On first connect, Claude Desktop will open a browser tab for Google login. Sign in with your `@customer-acquisition.co` account. After login, the MCP tools will appear in Claude.

---

## What the login flow looks like

You only need to log in once per day (the token expires every ~24 hours).

```
Claude Desktop / Cursor
    │
    │  1. Connects to MCP server → gets a 401 (no token yet)
    │
    ▼
Browser opens → Cloudflare Access Google login page
    │
    │  2. You sign in with your @customer-acquisition.co account
    │
    ▼
Token issued → stored locally → Claude/Cursor uses it for all tool calls
```

After login, every tool call goes:

```
Your machine → Twingate → MCP Server (K8s) → Laravel API → response back
```

The Cloudflare login page is not involved in tool calls at all.

---

## Troubleshooting

### Twingate connected but curl still fails

- Make sure you are connected to the correct Twingate network (not just any network)
- Ask your admin to confirm the resource `dashboard-mcp-stage-service.dashboard-api-staging` is provisioned and your account has access

### Claude Desktop: MCP server not showing up (no error, just missing)

Node.js is probably not found by Claude Desktop even if it works in your terminal. Fix: use the full path to `npx.cmd` in the config.

Find the path:
```cmd
where npx.cmd
```
Example output: `C:\Program Files\nodejs\npx.cmd`

Update your config:
```json
{
  "mcpServers": {
    "ops-mcp-stage-claude": {
      "command": "C:\\Program Files\\nodejs\\npx.cmd",
      "args": [
        "-y",
        "mcp-remote",
        "http://dashboard-mcp-stage-service.dashboard-api-staging/",
        "--allow-http"
      ]
    }
  }
}
```

### Claude Desktop: `EADDRINUSE: address already in use 127.0.0.1:31738`

A previous `mcp-remote` process is still running and holding the port. Kill it:

```cmd
# Find the PID using that port
netstat -ano | findstr :31738

# Kill it (replace 12345 with the actual PID)
taskkill /PID 12345 /F
```

Then fully quit Claude from the system tray and reopen.

If this keeps happening, clear the mcp-remote auth cache:

```cmd
# Windows
rmdir /s /q %APPDATA%\mcp-remote

# Mac
rm -rf ~/.mcp-remote
```

### Claude Desktop: browser opened for login but nothing happened / error

- Make sure you are signing in with your `@customer-acquisition.co` Google account
- Non-company Google accounts are blocked by Cloudflare Access

### Tools show up but return errors

- Twingate session may have expired — reconnect Twingate and retry
- Your CF login token may have expired — the browser will reopen automatically for re-login

---

## Quick reference

| Client | Config type | Node.js needed | Auth |
|---|---|---|---|
| Cursor | `"url"` only in mcp.json | No | Google SSO via browser (auto) |
| Claude Desktop | `"command": "npx.cmd"` + args | Yes | Google SSO via browser (auto) |

Both clients require Twingate to be running and your account to have Cloudflare Access permission.
