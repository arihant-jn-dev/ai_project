# OPS Automation via AI — All Approaches Explained

> Detailed guide on how to automate OPS interface operations (offers, bids, budgets) using AI — covering every viable approach, their flows, real examples, complexity, limitations, and scalability.

---

## The Problem We Are Solving

**Current painful workflow:**
```
Go to OPS Interface
  → Search for offer
    → Open offer
      → Edit (bid / budget / status / etc.)
        → Save Draft
          → Final Save
```

This is 5–7 manual steps for every single change. If you need to update 10 offers, that's 50–70 clicks across different pages.

**What we want:**
```
You type: "Pause offer XYZ and set bid to $1.50"
AI does: All 5–7 steps automatically
You get: "Done. Offer XYZ paused, bid updated to $1.50"
```

---

## What Your OPS System Must Support

Before choosing any approach, one thing is non-negotiable: **your OPS system must expose a way to be controlled programmatically**. This is either:

| Option | What It Means | Likelihood |
|---|---|---|
| **REST API** | OPS system has HTTP endpoints (`GET /offers`, `POST /offers/update`, etc.) | Most modern OPS tools have this |
| **GraphQL API** | Same idea, different query style | Less common but possible |
| **Database Access** | Direct read/write to the OPS database | Risky but possible for internal systems |
| **Browser Automation** | No API — a bot mimics clicking through the UI | Last resort, fragile |

**If your OPS system has an API → use Approaches 1–4 (clean, reliable)**
**If your OPS system has no API → use Approach 5 (browser automation, fragile)**

---

## Approach 1 — Slack Bot + AI Agent + MCP Server (The Flow You Described)

### What This Is

You connect a Slack Bot to an AI Agent (like OpenClaw or a custom agent). The agent understands your command, figures out what OPS actions to take, and calls your OPS system through an MCP server layer.

### Full Architecture

```
You (in Slack)
  ↓ "Update offer 123 bid to $2.00 and increase budget by 20%"
Slack (receives your message)
  ↓ Forwards to your Bot (via Events API / Webhook)
Slack Bot (Python/Node app listening for messages)
  ↓ Sends message text to Agent
AI Agent (OpenClaw / Custom LLM Agent)
  ↓ Understands intent → decides which tools to call
MCP Server (your OPS backend wrapper)
  ↓ Translates tool calls into actual OPS API calls
OPS System API
  ↓ Makes the change
MCP Server returns result
  ↓
AI Agent summarizes result
  ↓
Slack Bot posts reply in your channel
  ↓
You see: "✅ Done. Offer 123 bid updated to $2.00. Budget increased from $1000 to $1200."
```

### Step-by-Step Flow with Example

**You type in Slack:**
> `@ops-bot update offer 456 — set bid to $1.75, pause it, set daily budget $500`

**Step 1 — Slack receives the message**
Slack's Events API sends a webhook POST to your bot server with the message text and who sent it.

**Step 2 — Bot forwards to the Agent**
The bot strips out the `@ops-bot` mention and sends the raw text to the AI Agent via API call.

**Step 3 — Agent understands the intent**
The AI Agent (with your OPS system context injected as a system prompt) breaks down the command:
```json
{
  "actions": [
    { "tool": "update_offer_bid", "offer_id": "456", "bid": 1.75 },
    { "tool": "pause_offer", "offer_id": "456" },
    { "tool": "set_daily_budget", "offer_id": "456", "budget": 500 }
  ]
}
```

**Step 4 — MCP Server executes each action**
The MCP server receives each tool call and makes the corresponding OPS API call:
```
POST /api/offers/456/bid     { "bid": 1.75 }
POST /api/offers/456/status  { "status": "paused" }
POST /api/offers/456/budget  { "daily_budget": 500 }
```

**Step 5 — Results come back**
```
Offer 456: bid updated ✅
Offer 456: paused ✅
Offer 456: daily budget set to $500 ✅
```

**Step 6 — Slack reply**
> `✅ All done! Offer 456 — bid set to $1.75, status paused, daily budget $500. Changes saved.`

---

### What Is OpenClaw?

OpenClaw is an AI agent framework that:
- Connects to LLMs (Claude, GPT, etc.)
- Has built-in tool-calling (can call APIs, MCP tools, etc.)
- Handles multi-step reasoning ("first do X, then do Y, then Z")
- Manages conversation memory

You can also **replace OpenClaw with your own custom agent** — a Python script that calls Claude/GPT and passes tool definitions. OpenClaw just saves you from writing that from scratch.

---

### What Is the MCP Server Layer?

The MCP server is a wrapper you write around your OPS system's API. It exposes "tools" that the AI can call:

| MCP Tool | What It Does Under the Hood |
|---|---|
| `get_offer(offer_id)` | `GET /api/offers/{id}` |
| `update_bid(offer_id, bid)` | `POST /api/offers/{id}/bid` |
| `pause_offer(offer_id)` | `POST /api/offers/{id}/status { status: paused }` |
| `set_budget(offer_id, amount)` | `POST /api/offers/{id}/budget` |
| `search_offers(name, status)` | `GET /api/offers?name=X&status=Y` |
| `bulk_update_offers(filters, changes)` | Loops through filtered offers and applies changes |

The AI agent calls these tools by name with parameters. The MCP server handles the actual HTTP calls to your OPS system.

---

### Complexity

| Dimension | Rating | Notes |
|---|---|---|
| **Initial Setup** | High | Need Slack Bot + AI Agent + MCP Server — 3 components to build and connect |
| **Ongoing Maintenance** | Medium | If OPS API changes, update MCP tool only (1 place) |
| **Learning Curve** | Medium | MCP is well-documented; Slack Bot setup is standard |
| **Time to Build** | 2–4 weeks for a solid version | Slack Bot: 2 days, MCP tools: 1 week, Agent tuning: 1 week |

### Limitations

- **Slack dependency** — If Slack is down, the whole thing is down
- **Ambiguous commands** — "update the big offer" is vague; the AI may ask back or pick wrong
- **Multi-turn confirmations** — If you want "confirm before saving," handling that dialog in Slack is tricky
- **No visual preview** — You can't see what the offer looked like before/after in Slack easily
- **Rate limits** — Slack API, OPS API, and LLM API all have rate limits to manage
- **OPS API must exist** — If your OPS system has no API, this won't work

### Scalability

- **Single user → Team of 10:** Works great. Each Slack message is independent.
- **Bulk operations:** Agent can loop through hundreds of offers ("pause all offers with CPA > $50")
- **Multiple OPS systems:** Add more MCP tools; agent figures out which to use
- **Multi-tenant:** Harder — need to scope each user's permissions and OPS access

### Best For
Teams already on Slack who want a conversational, low-friction interface. Best when you trust the AI to make changes without heavy confirmation flows.

---

## Approach 2 — Custom Web Chat Interface + AI Agent (Like Your Analytics Agent)

### What This Is

Build a web chat interface (like the analytics agent you already have) but for OPS operations. You type commands in a browser chat, and the agent executes them on your OPS system.

### Full Architecture

```
You (in Browser)
  ↓ "Pause all offers where yesterday's CPA > $60"
Web Chat UI (like your analytics agent's index.html)
  ↓ POST /api/chat  { query: "..." }
Flask Web Server (app.py)
  ↓
OPS Agent (similar to InteractiveMetricsAnalysisAgent)
  ↓ LLM understands command → decides tool calls
OPS Tools (wrappers around OPS API)
  ↓ Calls OPS API
OPS System
  ↓ Returns results
Agent summarizes + displays
  ↓
Web UI shows: "Paused 12 offers. Here's the list..."
```

### Step-by-Step Flow with Example

**You type:**
> `Show me all active offers with daily budget under $200, then pause the ones where CPA is above $50`

**Step 1 — Agent understands intent**
LLM breaks this into a two-step plan:
1. Fetch all active offers with budget < $200
2. For each, check CPA; if > $50, pause it

**Step 2 — Fetch offers**
Tool call: `search_offers(status="active", max_budget=200)`
Returns: 18 offers

**Step 3 — Check CPA and filter**
Tool call: `get_offer_performance(offer_ids=[...], metric="cpa", date="yesterday")`
Returns: 7 offers have CPA > $50

**Step 4 — Pause those 7**
Tool call: `pause_offers(offer_ids=[...])`
Returns: All 7 paused successfully

**Step 5 — Show result in chat**
```
Found 18 active offers with budget < $200.
7 had CPA > $50 yesterday — paused all of them.

Paused offers:
• Offer 101 — CPA $67, Budget $150
• Offer 234 — CPA $82, Budget $180
• Offer 389 — CPA $55, Budget $120
...

Remaining 11 offers unchanged (CPA within limit).
```

---

### How This Differs From Approach 1

| | Slack Bot (Approach 1) | Web Chat (Approach 2) |
|---|---|---|
| Interface | Slack | Browser |
| Requires Slack | Yes | No |
| Can show tables/charts | Limited | Full HTML tables, charts |
| Confirmation flows | Awkward in Slack | Easy — show a confirm button |
| "Before/after" preview | Hard | Easy — show side-by-side |
| Mobile-friendly | Yes (Slack app) | Yes (responsive design) |
| Access control | Slack workspace | Login system (Cloudflare, etc.) |

### Complexity

| Dimension | Rating | Notes |
|---|---|---|
| **Initial Setup** | Medium | You already have the pattern from your analytics agent |
| **Reuse from Analytics Agent** | High | Same Flask + LLM + tool pattern, just different tools |
| **Time to Build** | 1–2 weeks | Most scaffolding exists already |

### Limitations

- **Need to open a browser** — slightly more friction than Slack (which you're already in)
- **Session management** — if you close the tab, context is lost (solvable with backend sessions)
- **No notifications** — unlike Slack, the chat won't ping you when a scheduled job finishes

### Scalability

- Easy to scale with standard web app scaling
- Can add user authentication, per-user history, audit logs
- Rich UI means you can add dashboards, not just chat

### Best For
Teams who want a dedicated OPS command center, rich outputs (tables, diffs, charts), and don't want to depend on Slack. Also the fastest to build given your existing analytics agent codebase.

---

## Approach 3 — Claude Desktop + MCP Server (No Slack, No Custom App)

### What This Is

You already built this pattern with your analytics agent. Apply the same thing to OPS. Claude Desktop connects directly to an OPS MCP server. You talk to Claude like normal, and it executes OPS operations.

### Full Architecture

```
Claude Desktop (your laptop)
  ↓ "Pause offer 456 and set bid to $1.50"
MCP Protocol (over HTTP or stdio)
  ↓
OPS MCP Server (running locally or on a server)
  ↓ Translates tool calls → OPS API calls
OPS System
  ↓ Makes changes
MCP Server returns result
  ↓
Claude Desktop shows: "Done. Offer 456 paused, bid set to $1.50"
```

### Step-by-Step Flow with Example

1. You open Claude Desktop on your computer
2. Claude has the `ops_agent` MCP tool available (shown by 🔌 icon)
3. You type: *"Find all offers with status active and budget over $1000, then list them"*
4. Claude calls `search_offers(status="active", min_budget=1000)` via MCP
5. OPS MCP server hits your OPS API
6. Claude displays the list in a clean, readable format
7. You say: *"Pause the top 3 by spend"*
8. Claude calls `pause_offer()` three times
9. Claude confirms: *"Paused offer 101, 234, 389. All changes saved."*

---

### What MCP Tools You Would Build

```python
# Tools your OPS MCP server would expose:

get_offer(offer_id)              # Fetch single offer details
search_offers(filters)           # Search/filter offers
create_offer(params)             # Create new offer
update_offer_bid(offer_id, bid)  # Change bid
update_offer_budget(offer_id, budget)  # Change budget
pause_offer(offer_id)            # Pause
resume_offer(offer_id)           # Resume
bulk_pause(filters)              # Pause all matching
bulk_update_bids(filters, bid)   # Update bids in bulk
get_offer_performance(offer_id)  # CPA, CVR, spend, etc.
```

### Complexity

| Dimension | Rating | Notes |
|---|---|---|
| **Initial Setup** | Low-Medium | You already know MCP from the analytics agent |
| **No custom UI needed** | ✅ | Claude Desktop IS the UI |
| **Time to Build** | 3–5 days | Just the MCP server + OPS tool wrappers |
| **Pattern Reuse** | Very High | Literally copy the MCP server pattern from analytics agent |

### Limitations

- **Desktop app only** — Works on your machine; not great for sharing with a team
- **Claude Desktop is required** — Not everyone has it configured
- **No audit trail by default** — You need to add logging if you want a record of who changed what
- **No custom UI** — Can't add confirmation buttons, visual diffs, etc.
- **Context limits** — If you paste a huge list of offers, Claude's context window fills up

### Scalability

- Great for 1–3 power users
- Not ideal for teams of 10+ (each person needs their own Claude Desktop setup)
- Can be made "remote" (like your analytics agent) so team shares one deployed MCP server

### Best For
Personal power-user setup. If you are the main person making OPS changes, this is the fastest to get running (days, not weeks) and requires zero custom UI. Just build the MCP server and connect it to Claude Desktop.

---

## Approach 4 — n8n / Make.com Workflow Automation + AI

### What This Is

Use a no-code/low-code workflow tool (n8n, Make.com, Zapier) to create automated OPS workflows triggered by natural language inputs (via a Slack command, form, email, etc.).

### Full Architecture

```
Trigger (Slack Command / Form / Schedule)
  ↓
n8n Workflow
  ↓ AI Node (calls Claude/GPT to parse the command)
  ↓ Conditional branches (what action was requested?)
  ├── If "pause offer" → HTTP Node → POST /api/offers/{id}/pause
  ├── If "update bid"  → HTTP Node → POST /api/offers/{id}/bid
  ├── If "create offer" → HTTP Node → POST /api/offers
  └── If "bulk update" → Loop Node → Multiple HTTP calls
  ↓
Notification Node (Slack / Email reply)
  ↓
"Done. Changes applied."
```

### Example Workflow in n8n

**Workflow: "Daily Budget Optimizer"**

1. **Trigger:** Every day at 8am
2. **HTTP Node:** `GET /api/offers?status=active` — get all active offers
3. **HTTP Node:** `GET /api/performance?date=yesterday` — get yesterday's CPA for each
4. **AI Node (Claude):** "Given this list of offers and their CPAs, which ones should have their bids reduced by 10% (CPA > $60), kept the same (CPA $40–60), or increased by 10% (CPA < $40)? Return JSON."
5. **Switch Node:** Routes each offer to the right action
6. **HTTP Nodes:** Makes the bid updates
7. **Slack Node:** Posts summary to `#ops-updates` channel

**You don't write any code.** You drag and drop nodes in n8n's visual editor.

---

### Complexity

| Dimension | Rating | Notes |
|---|---|---|
| **Initial Setup** | Low | n8n has a UI, no coding required for basic flows |
| **Complex Logic** | Medium-High | Multi-step conditional flows get complex visually |
| **AI Integration** | Medium | n8n has AI nodes; Make.com requires more setup |
| **OPS API Integration** | Low | HTTP nodes for any REST API |
| **Time to Build** | 1–3 days for simple flows | Complex bulk operations: 1 week |

### Limitations

- **Rigid flows** — You define workflows upfront; hard to handle truly free-form commands ("do something unusual")
- **AI parsing can fail** — If the AI misinterprets the command, the wrong workflow fires
- **Debugging is visual but tricky** — Large workflows become hard to maintain
- **Cost** — n8n cloud, Make.com, etc. have monthly fees; self-hosted n8n is free
- **Not conversational** — Each trigger is independent; no "follow-up" capability
- **Schema changes break flows** — If OPS API changes a field name, your flows break

### Scalability

- Great for repetitive, structured tasks ("every morning, optimize bids")
- Poor for free-form, one-off commands
- Scales well for scheduled automation
- Not great as the primary interface for ad-hoc changes

### Best For
Scheduled, repetitive automation (daily bid optimization, weekly budget resets, nightly reporting). Works alongside Approach 1 or 2 — use n8n for scheduled tasks, use the AI chat for ad-hoc requests.

---

## Approach 5 — Browser Automation (Playwright/Selenium) + AI

### What This Is

If your OPS system has **no API**, a bot can control the browser — literally clicking through the OPS interface automatically, just like a human would. The AI interprets your command and controls the browser.

### Full Architecture

```
You type: "Pause offer 456"
  ↓
AI Agent understands intent → calls browser automation tool
  ↓
Playwright/Selenium (controls a real browser)
  ↓ Opens OPS URL in browser
  ↓ Clicks "Search"
  ↓ Types "456" in search box
  ↓ Clicks on the offer
  ↓ Finds the "Status" dropdown
  ↓ Selects "Paused"
  ↓ Clicks "Save"
  ↓
Browser automation confirms change
  ↓
You see: "Done. Offer 456 paused."
```

### Tools That Make This Easier

| Tool | What It Is |
|---|---|
| **Playwright** | Microsoft's browser automation library (Python/JS) |
| **Selenium** | Older browser automation, widely supported |
| **Browser Use** | AI-native browser automation (LLM + Playwright together) |
| **Skyvern** | AI agent specifically for browser automation of web UIs |
| **Stagehand** | Newer AI browser framework by Browserbase |

The modern approach uses an LLM (Claude/GPT) that can see the browser screen (as a screenshot or HTML) and decide what to click next — you don't need to hardcode every click.

---

### Complexity

| Dimension | Rating | Notes |
|---|---|---|
| **Initial Setup** | High | Need to map out the OPS UI steps, handle page loads, popups |
| **Maintenance** | Very High | Every UI change in OPS breaks your automation |
| **Reliability** | Low-Medium | UI automation is fragile — CAPTCHAs, popups, slow loads all cause failures |
| **Time to Build** | 2–4 weeks | Mapping full OPS UI flows is tedious |

### Limitations

- **Extremely fragile** — OPS team changes a button label or page layout → your bot breaks
- **Slow** — Loading full browser pages is 5–10× slower than API calls
- **Hard to debug** — Race conditions, timeouts, element-not-found errors
- **No bulk operations** — Each action needs a separate browser journey
- **Requires a running browser** — Needs a server with a display (or headless mode)
- **Login sessions expire** — Need to handle re-login, 2FA, session cookies

### Scalability

- Poor. Each parallel operation needs its own browser instance.
- Fine for 1 action at a time; problematic for bulk updates of 100+ offers

### Best For
**Only use this if your OPS system has absolutely no API.** This is a last resort. If there is any API access, avoid this approach.

---

## Approach 6 — Telegram / WhatsApp Bot + AI Agent

### What This Is

Instead of Slack, use Telegram or WhatsApp as the conversational interface. Both have bot APIs that are easy to set up and work on mobile.

### Full Architecture

```
You (in Telegram / WhatsApp)
  ↓ "Update bid for offer 456 to $2.00"
Telegram Bot API / WhatsApp Business API
  ↓
Webhook → Your Bot Server
  ↓
AI Agent (Claude/GPT)
  ↓
OPS API
  ↓
Reply in Telegram/WhatsApp
```

### Why Consider This Over Slack?

| | Slack | Telegram | WhatsApp |
|---|---|---|---|
| Cost | Paid (for large teams) | Free | Free / WhatsApp Business API cost |
| Mobile UX | Good | Excellent | Excellent |
| Setup complexity | Medium | Low | Medium |
| Bot API | Good | Excellent (very simple) | More complex |
| Team use | Best for teams | Good for small groups | Good for small groups |
| File sharing | Good | Good | Good |

Telegram is the simplest to build a bot on — the Bot API is free, well-documented, and has no approval process. WhatsApp requires Meta Business API approval which takes longer.

### Complexity

| Dimension | Rating | Notes |
|---|---|---|
| **Telegram Bot Setup** | Very Low | Create a bot with BotFather, get a token, done |
| **AI Integration** | Same as Slack | Same LLM agent pattern |
| **Time to Build** | 1–2 weeks | Easier than Slack in some ways |

### Limitations
- Same as Slack approach but without the enterprise features
- WhatsApp has strict messaging policy limits
- Not great for team collaboration (no channels like Slack)

### Best For
If you/your team uses Telegram or WhatsApp more than Slack. Also great as a personal OPS bot for individual use.

---

## Approach 7 — Voice Interface + AI Agent (Experimental)

### What This Is

Connect a voice interface (phone call, Siri Shortcuts, or a web voice UI) to your AI agent. You speak the command, AI converts it to text, agent executes the OPS action, and speaks the result back.

### Example

You press a shortcut on your phone and say:
> *"Pause all offers with budget over a thousand dollars"*

AI hears → converts to text → agent executes → you hear:
> *"Done. 5 offers paused. Offer 101, 234, 389, 456, and 512."*

### Complexity: High. Reliability: Medium. Coolness factor: Very High.

### Best For
Very early prototype / demo. Not production-ready for complex OPS operations. Useful for simple, frequent single commands ("pause offer X", "check status of offer Y").

---

## Side-by-Side Comparison of All Approaches

| | Approach 1: Slack+MCP | Approach 2: Web Chat | Approach 3: Claude Desktop | Approach 4: n8n | Approach 5: Browser Bot | Approach 6: Telegram |
|---|---|---|---|---|---|---|
| **Interface** | Slack | Browser | Claude Desktop | Visual flow editor | Browser (automated) | Telegram/WhatsApp |
| **Requires OPS API** | Yes | Yes | Yes | Yes | No (UI fallback) | Yes |
| **Conversational** | Yes | Yes | Yes | No | No | Yes |
| **Ad-hoc commands** | Excellent | Excellent | Excellent | Poor | Medium | Excellent |
| **Scheduled automation** | Medium | Poor | Poor | Excellent | Medium | Medium |
| **Bulk operations** | Good | Excellent | Good | Good | Poor | Good |
| **Rich UI (tables, charts)** | Limited | Excellent | None | None | None | Limited |
| **Team use** | Excellent | Good | Poor | Good | Medium | Medium |
| **Setup complexity** | High | Medium | Low | Low | Very High | Low |
| **Maintenance effort** | Medium | Medium | Low | Medium | Very High | Medium |
| **Time to build** | 3–4 weeks | 1–2 weeks | 3–5 days | 1–3 days | 3–4 weeks | 1–2 weeks |
| **Cost** | Slack fees | Server hosting | Claude subscription | n8n fees | Server costs | Free |
| **Reliability** | High | High | High | High | Low | High |
| **Scalability** | High | High | Medium | High | Low | Medium |

---

## Our Recommended Approach

### For You Right Now — Start With Approach 3 (Claude Desktop + MCP)

**Why:** You already built this exact pattern with the analytics agent. The MCP server pattern is already in your codebase. You just need to build OPS-specific tools. Fastest path from today to working solution.

**Time to working prototype:** 3–5 days

**What you build:**
- An OPS MCP server (just like `mcp_server.py` in your analytics agent)
- Tool definitions for: get_offer, update_bid, update_budget, pause_offer, resume_offer, search_offers
- Each tool calls your OPS API

**Then connect it to Claude Desktop** — same steps as `MCP_SETUP.md` in this repo.

---

### Phase 2 — Add Web Chat (Approach 2)

Once you have the MCP tools working, wrap them in a web chat interface (copy the pattern from your analytics agent's `app.py` + `index.html`). This gives your team a dedicated OPS command center without needing Claude Desktop.

**Time:** 1–2 weeks

**What you get:**
- Browser-based chat for OPS commands
- Rich output (tables showing offer lists, before/after changes)
- Confirmation flows before destructive actions ("pausing 50 offers — confirm?")
- Audit log of all changes made

---

### Phase 3 — Add Slack Bot (Approach 1)

Once the core agent works, add Slack as a second interface. The Slack bot just relays messages to the same backend. This is now only 2–3 days of work since the hard part (the agent + MCP tools) is already done.

---

### For Scheduled Automation — Add n8n (Approach 4)

Run n8n alongside for scheduled tasks:
- Daily: Optimize bids based on yesterday's CPA
- Weekly: Reset budgets
- Nightly: Pause underperforming offers

These are too rigid for conversational AI but perfect for visual workflow automation.

---

## What You Need to Build (Checklist)

### Phase 1 — OPS MCP Server (Days 1–5)

- [ ] Get OPS system API documentation
- [ ] List all operations you need: create offer, update bid, update budget, pause, resume, search, bulk operations
- [ ] Write MCP tool definitions (one function per operation)
- [ ] Each function calls the OPS REST API
- [ ] Add authentication (API key or OAuth token for OPS system)
- [ ] Test each tool individually
- [ ] Connect to Claude Desktop (copy `MCP_SETUP.md` steps)
- [ ] Test with real commands

### Phase 2 — Web Chat (Days 6–15)

- [ ] Create OPS Agent (similar to `interactive_profit_agent.py` but for OPS)
- [ ] Write system prompt for OPS context (what operations are available, how to handle ambiguity)
- [ ] Add confirmation flow for destructive actions (bulk pause, bulk delete)
- [ ] Copy `index.html` from analytics agent, update for OPS branding
- [ ] Add offer list display (table format)
- [ ] Add audit log (who changed what, when)
- [ ] Deploy (same Docker setup as analytics agent)

### Phase 3 — Slack Integration (Days 16–20)

- [ ] Create Slack App at api.slack.com
- [ ] Set up Events API webhook
- [ ] Write bot server (listens for messages, calls OPS Agent)
- [ ] Handle @mentions and direct messages
- [ ] Add `/ops` slash command for quick actions
- [ ] Test in your Slack workspace

---

## What the OPS Agent System Prompt Should Look Like

This is the context you inject into Claude so it knows what it can do:

```
You are an OPS management assistant with access to the following tools:

OFFER MANAGEMENT:
- get_offer(offer_id) — Fetch details of a specific offer
- search_offers(status, min_budget, max_budget, name) — Search offers with filters
- create_offer(name, bid, budget, targeting) — Create a new offer
- update_offer(offer_id, fields) — Update offer fields

BID MANAGEMENT:
- update_bid(offer_id, bid) — Set a new bid for an offer
- bulk_update_bids(offer_ids, bid) — Update bids for multiple offers

BUDGET MANAGEMENT:
- set_daily_budget(offer_id, amount) — Set daily budget
- set_total_budget(offer_id, amount) — Set total/lifetime budget
- adjust_budget_percent(offer_id, percent) — Increase/decrease budget by %

STATUS MANAGEMENT:
- pause_offer(offer_id) — Pause a single offer
- resume_offer(offer_id) — Resume a paused offer
- bulk_pause(filters) — Pause all offers matching criteria
- bulk_resume(filters) — Resume all offers matching criteria

BUSINESS RULES:
- Always confirm before bulk operations affecting more than 5 offers
- Never reduce a bid below $0.50
- Never set a daily budget below $100
- When a user says "pause low performers," define low performer as CPA > $60 unless specified
- Always show a summary of changes made after each operation

RESPONSE FORMAT:
- For single offer changes: Confirm the change with before/after values
- For bulk operations: Show count + list of affected offers
- If something fails: Explain clearly what failed and why
```

---

## Example Conversations (What This Looks Like in Practice)

### Example 1 — Simple Bid Update

**You:** Update bid for offer 789 to $1.80

**Agent:** 
> Updated. Offer 789 (Campaign: Summer Leads)
> Bid: $2.10 → **$1.80**
> Status: Active | Daily Budget: $800
> Change saved successfully.

---

### Example 2 — Bulk Action with Confirmation

**You:** Pause all offers where yesterday CPA was above $70

**Agent:**
> Found 8 offers with CPA > $70 yesterday:
>
> | Offer | Name | CPA | Budget |
> |---|---|---|---|
> | 101 | NYC Campaign | $82 | $500 |
> | 234 | West Coast | $71 | $1200 |
> | 389 | Mobile Banner | $95 | $300 |
> | ... | | | |
>
> ⚠️ This will pause 8 offers with combined daily budget of $4,800.
> **Type "confirm" to proceed or "cancel" to abort.**

**You:** confirm

**Agent:**
> ✅ Paused 8 offers successfully.
> Combined budget freed: $4,800/day

---

### Example 3 — Natural Language Search

**You:** Show me all active offers on Google traffic with bid between $1 and $2

**Agent:**
> Found 14 active offers on Google traffic, bid $1–$2:
>
> | Offer | Bid | Daily Budget | Yesterday CPA |
> |---|---|---|---|
> | 201 | $1.50 | $600 | $42 |
> | 345 | $1.80 | $900 | $67 |
> | 567 | $1.20 | $400 | $38 |
> | ... | | | |

---

### Example 4 — Complex Multi-Step

**You:** For all Google offers with CPA under $45, increase bid by 10%. For offers with CPA over $65, decrease bid by 10%. Leave others unchanged.

**Agent:**
> Analyzing 14 Google offers from yesterday...
>
> **Bid increases (CPA < $45) — 6 offers:**
> 201: $1.50 → $1.65 | 567: $1.20 → $1.32 | ...
>
> **Bid decreases (CPA > $65) — 3 offers:**
> 345: $1.80 → $1.62 | ...
>
> **No change (CPA $45–$65) — 5 offers**
>
> Total: 9 bids updated. Confirm?

---

## Summary

| | What To Do |
|---|---|
| **This week** | Build OPS MCP server + connect to Claude Desktop |
| **This month** | Build web chat interface for team use |
| **Next quarter** | Add Slack bot + n8n for scheduled automation |
| **If no OPS API** | Request API access from OPS team first; browser automation is last resort |
| **OpenClaw** | Use it only if you need an off-the-shelf agent framework; you can build the same thing custom using Claude's tool-calling API |

---

*Created based on your current OPS workflow requirements and the MCP/AI patterns already established in your analytics agent.*
