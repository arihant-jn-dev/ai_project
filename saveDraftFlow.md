# Save-Draft Flow — Complete Reference

This document explains how an offer update travels from a user's natural-language
request all the way through the MCP server, into a saved draft, and out to a
staging console URL. Every function, tool, and decision point is covered.

---

## Table of Contents

0. [LLM vs MCP Server — Who Decides What?](#0-llm-vs-mcp-server--who-decides-what)
1. [Why Draft-Only?](#1-why-draft-only)
2. [The Two-Phase Pattern](#2-the-two-phase-pattern)
3. [How the LLM Decides Which Tool to Call](#3-how-the-llm-decides-which-tool-to-call)
4. [Full Flow — Step by Step](#4-full-flow--step-by-step)
5. [Tool: `update_offer` (Phase 1 — Preview)](#5-tool-update_offer-phase-1--preview)
6. [Tool: `confirm_offer_update` (Phase 2 — Save Draft)](#6-tool-confirm_offer_update-phase-2--save-draft)
7. [Internal Helper: `_build_offer_update_payload`](#7-internal-helper-_build_offer_update_payload)
8. [Internal Helper: `_make_confirmation_response`](#8-internal-helper-_make_confirmation_response)
9. [ActivityLogService — Diffing Two Snapshots](#9-activitylogservice--diffing-two-snapshots)
10. [offer_filter — executeSteps Computation](#10-offer_filter--executesteps-computation)
11. [api.save_draft — The HTTP Call](#11-apisave_draft--the-http-call)
12. [Console URL Construction](#12-console-url-construction)
13. [Auth — Who Makes the API Call?](#13-auth--who-makes-the-api-call)
14. [Worked Example: Bid 1.7 → 1.8](#14-worked-example-bid-17--18)
15. [What Is Deliberately Disabled](#15-what-is-deliberately-disabled)
16. [Data Flow Diagram](#16-data-flow-diagram)

---

## 0. LLM vs MCP Server — Who Decides What?

There are exactly **two brains** in this system. Understanding which one is
responsible for which decision is the key to understanding the whole flow.

| Brain | Role |
|---|---|
| **LLM** (Claude / GPT / any model) | Reads English, understands user intent, decides which tool to call and with what arguments. Never touches the API directly. |
| **MCP Server** (Python — `server.py`) | Executes the tool, validates inputs, calls the Laravel API, returns structured JSON. Never understands English. |

The **tool descriptions** are the contract between them. The LLM reads every
tool's `name` + `description` at connection time and uses them as its only
rulebook. This is why routing rules, clarification rules, and caching rules are
written directly into the description strings — not in code.

---

### The Decision Loop (every user message)

```
User types something
        │
        ▼
┌───────────────┐
│      LLM      │  reads all tool descriptions (loaded once at connect)
│               │
│  1. Understand intent
│  2. Do I have enough info?
│     ├── No → ask user for clarification  (no tool called)
│     └── Yes → pick a tool + fill args
└───────┬───────┘
        │ tool call: name + arguments (JSON)
        ▼
┌───────────────────┐
│    MCP Server     │  pure Python — no language understanding
│                   │
│  1. Validate inputs  (validation.py)
│     ├── invalid → return {"success": false, "error": "..."}
│     └── valid → continue
│  2. Call Laravel API  (api.py)
│  3. Process response  (offer_filter, activity_log, utility)
│  4. Return JSON dict
└───────┬───────────┘
        │ JSON response
        ▼
┌───────────────┐
│      LLM      │  reads the JSON, formats a human-readable reply
└───────────────┘
        │
        ▼
  User sees response
```

---

### Decision Ownership Table

| Decision | Owner | Where it lives |
|---|---|---|
| Which tool to call | LLM | Tool `description` strings |
| When NOT to call a tool (use context instead) | LLM | Tool `description` strings |
| Ask user for clarification before calling | LLM | Tool `description` strings |
| Validate input values (bid > 0, budget format, etc.) | MCP Server | `validation.py` |
| Fetch data from the API | MCP Server | `api.py` |
| Compute executeSteps / activityLog | MCP Server | `offer_filter.py`, `activity_log.py` |
| Format the response for the user | LLM | LLM's own reasoning |

---

### Example 1 — User gives an offer name, not a UUID

```
User: "show me details of angular test offer"
```

**LLM decides** (by reading `get_offer` description):
> "UUID required. If name given and UUID not in context → call list_offers first."

```
LLM → list_offers()
MCP Server → GET /api/admin/offer/get-offer-list → returns all offers
LLM → scans list, finds "angular test - again - cloned" → UUID = ec140104-...
LLM → get_offer(offer_unique_id="ec140104-...")
MCP Server → GET /api/admin/offer/get-offer/ec140104-... → returns full offer
LLM → formats and shows details to user
```

Two tool calls happened. The LLM decided both — the MCP server just executed them.

---

### Example 2 — Same offer asked again in the same conversation

```
User: "what's the bid on that offer?"
  (offer was already fetched 3 messages ago)
```

**LLM decides** (by reading `get_offer` description):
> "If full offer details were already fetched this conversation, answer from
> context — do NOT call this tool again."

```
LLM → answers directly: "The bid is $1.70 (CPL)"
```

Zero tool calls. The MCP server is never involved.

---

### Example 3 — Ambiguous field ("budget")

```
User: "update budget to 5"
```

**LLM decides** (by reading `update_offer` description):
> "If user says 'budget' without specifying daily or overall → ask first."

```
LLM → asks: "Do you want to update the daily budget, the overall budget, or both?"
  (no tool called yet)

User: "daily"

LLM → update_offer(offer_unique_id="...", new_daily_budget="5")
MCP Server → validates "5" → valid positive number → continues
MCP Server → fetches live offer, computes diff, returns confirmationRequired=True
LLM → shows confirmation prompt to user
```

The LLM held back until it had enough information. The MCP server only ran
after the LLM was confident about the intent.

---

### Example 4 — Invalid input caught server-side

```
User: "set bid to -5"
```

**LLM decides**: user wants to update bid → calls `update_offer(new_offer_bid=-5.0)`

**MCP Server catches it** (`validation.py`):
```python
validate_offer_bid(-5.0)
→ {"success": False, "error": "bid must be a positive dollar amount (got -5.0)"}
```

```
MCP Server → returns error dict immediately (no API call made)
LLM → reads error, tells user: "That's not valid — bid must be a positive dollar amount"
```

The MCP server caught this one, not the LLM. This is intentional: the LLM
cannot always be trusted to reject bad values on its own (especially with a
different model), so server-side validation is the safety net.

---

### Example 5 — Full two-phase write flow

```
User: "update bid to 1.8 for ec140104-..."
```

```
LLM → update_offer(offer_unique_id="ec140104-...", new_offer_bid=1.8)

MCP Server:
  1. validate_offer_update_inputs(1.8, ...) → None (valid)
  2. GET /api/admin/offer/get-offer/ec140104-... → live offer (bid=1.7)
  3. Apply mutation: offerBid = 1.8
  4. ActivityLogService.diff(before, after) → [{fieldName: "offerBid", prev: 1.7, curr: 1.8}]
  5. evaluate_offer_steps() → ["LEADGEN", "MAX_CAMPAIGN", "MAX_ADGROUP", ...]
  6. Return confirmationRequired=True + confirmationPrompt

LLM → shows confirmation prompt verbatim to user

User: "yes"

LLM → confirm_offer_update(offer_unique_id="ec140104-...", new_offer_bid=1.8)

MCP Server:
  1. validate_offer_update_inputs(1.8, ...) → None (valid)
  2. Re-fetches live offer fresh (never uses stale state)
  3. Rebuilds full payload
  4. POST /api/admin/offer/save-draft-data → draft saved
  5. build_console_url("hostpost", "ec140104-...") → staging URL
  6. Returns {draftSaved: true, consoleUrl: "https://staging-console.../hostpost/ec140104-..."}

LLM → tells user: "Draft saved. Review here: https://staging-console.../hostpost/ec140104-..."
```

---

### The One Rule to Remember

> **The LLM decides WHEN and WHAT to call.**
> **The MCP server decides IF the input is valid and HOW to execute.**

Everything in the tool descriptions shapes LLM behaviour.
Everything in `validation.py`, `api.py`, and `offer_filter.py` shapes server behaviour.
The two never overlap.

---

## 1. Why Draft-Only?

All offer updates go through **draft**, never directly to live. This is intentional:

- A human must review changes in the staging console UI before they are promoted.
- The MCP server never calls `update-offer-complete` directly (that endpoint is
  commented out and disabled in `api.py`).
- The only write endpoint the MCP server calls is `POST /api/admin/offer/save-draft-data`.

The staging console URL returned after every save is the hand-off point: the MCP
server's job ends there. The user clicks the link, reviews the diff in the UI,
and clicks "Promote to Live" themselves.

---

## 2. The Two-Phase Pattern

Every write operation in this server follows a **preview → confirm** loop:

```
User request
     │
     ▼
[PHASE 1]  update_offer       ← NEVER writes anything
     │      • fetches live offer
     │      • checks for an existing draft
     │      • computes pending changes
     │      • returns confirmationPrompt to show to user
     ▼
User sees the diff + any existing-draft warning
     │
     ├─── "no" / "cancel" ──► nothing happens
     │
     └─── "yes" / "confirm"
               │
               ▼
         [PHASE 2]  confirm_offer_update    ← WRITES the draft
                     • re-fetches offer fresh (never uses stale state)
                     • rebuilds the full payload
                     • calls api.save_draft()
                     • returns staging console URL
```

The LLM **must not** call `confirm_offer_update` unless the user explicitly
replied "yes" or "confirm". If the user says "no" or "cancel", the flow stops
after Phase 1 and nothing is written.

---

## 3. How the LLM Decides Which Tool to Call

The LLM (Cursor) reads the `name` and `description` of every registered MCP tool
at connection time. The descriptions are the sole source of truth for routing
decisions — there is no hidden prompt or system instruction.

### Tool selection rules (from the descriptions)

| Situation | Tool selected | Reason in description |
|---|---|---|
| User asks anything about an offer (bid, status, targeting, etc.) | `get_offer` | "Use this for ANY question about an offer" |
| User wants to see a list of offers | `list_offers` | "List all advertising offers" |
| User wants to change a field on an offer | `update_offer` | "PHASE 1 — Preview an offer update … This tool NEVER writes" |
| User has just said "yes" after a Phase 1 preview | `confirm_offer_update` | "PHASE 2 — Save offer changes … Call this ONLY when … the user has explicitly replied 'yes' or 'confirm'" |
| User wants to verify auth or re-login after a 401 | `login` | "Call this tool explicitly if … you just had credentials rotated" |

### Why the LLM never skips Phase 1

The `confirm_offer_update` description explicitly says:

> "Call this ONLY when: 1. update_offer was called first and returned
> confirmationRequired=True, AND 2. The user has explicitly replied 'yes' or 'confirm'."

The LLM reads this constraint in the tool description and will not call
`confirm_offer_update` unless both conditions are met.

### What fields can be changed

The LLM infers which parameters to pass from the `update_offer` description:

| Parameter | Type | Example |
|---|---|---|
| `new_offer_bid` | float | `1.8` |
| `new_offer_name` | string | `"My Offer v2"` |
| `new_daily_budget` | string | `"500"` or `"UNLIMITED"` |
| `new_overall_budget` | string | `"2000"` or `"UNLIMITED"` |
| `new_offer_sub_type` | integer | `10` |

Only the parameters that need to change are passed; the rest default to `None`
and are ignored.

---

## 4. Full Flow — Step by Step

```
User: "update bid to 1.8 for offer ec140104-..."
         │
         ▼
LLM → calls update_offer(offer_unique_id="ec140104-...", new_offer_bid=1.8)
         │
         ▼
server.update_offer()
  1. Validates at least one field was supplied
  2. Calls _get_user_creds() → resolves auth token
  3. Calls _build_offer_update_payload()
       ├── api.get_offer() → GET /api/admin/offer/get-offer/{uuid}
       │     └── returns { response: { live: {...}, draft: {...} } }
       ├── _extract_live()  → pulls response.live
       ├── _extract_draft() → pulls response.draft  (None if no pending draft)
       ├── build_live_offer_for_draft(live) → deep-copy, strips offerActivityLog
       ├── Applies mutations: data["offerBid"] = 1.8
       ├── ActivityLogService.get_activity_logs(before, after)
       │     → [{ fieldName: "offerBid", previousValue: 1.7, currentValue: 1.8 }]
       ├── evaluate_offer_steps() → ["LEADGEN", "MAX_CAMPAIGN", "MAX_ADGROUP", "MAX_ADGROUP_TARGETTING"]
       ├── data["currentVersion"] += 0.01
       └── returns { success, data, live, existing_draft, changes, activity_log, execute_steps }
  4. Checks for existing_draft → builds warning text if found
  5. Calls _make_confirmation_response() → builds confirmationPrompt
  6. Returns to LLM
         │
         ▼
LLM shows user the confirmationPrompt verbatim:
  "Pending changes: offerBid: 1.7 → 1.8
   ⚠️ An existing draft was found (bid: $5). Saving will OVERWRITE it.
   Reply 'yes' or 'confirm' to save, or 'no'/'cancel' to abort."
         │
User: "yes go ahead"
         │
         ▼
LLM → calls confirm_offer_update(offer_unique_id="ec140104-...", new_offer_bid=1.8)
         │
         ▼
server.confirm_offer_update()
  1. Validates at least one field was supplied
  2. Calls _get_user_creds() again (fresh)
  3. Calls _build_offer_update_payload() again (re-fetches live — no stale state)
  4. Calls api.save_draft(settings, {"data": data})
       └── POST /api/admin/offer/save-draft-data
  5. Calls _build_console_url(offer_type, offer_unique_id)
       └── "https://staging-console.customer-acquisition.co/hostpost/ec140104-..."
  6. Returns { success, draftSaved, changes, executeSteps, activityLog, consoleUrl }
         │
         ▼
LLM shows user:
  "Draft saved. Review and promote: https://staging-console.../hostpost/ec140104-..."
```

---

## 5. Tool: `update_offer` (Phase 1 — Preview)

**File:** `mcp_server/server.py`

**Purpose:** Preview what would change. Never writes anything.

### Inputs

```python
offer_unique_id: str           # required — UUID of the offer
new_offer_bid: float | None    # optional — new bid in dollars
new_offer_name: str | None     # optional — new display name
new_daily_budget: str | None   # optional — "500" or "UNLIMITED"
new_overall_budget: str | None # optional — "2000" or "UNLIMITED"
new_offer_sub_type: int | None # optional — numeric sub-type ID
```

### What it does

1. **Guards:** Returns an error if no fields are provided (nothing to change).
2. **Resolves auth:** Calls `_get_user_creds()`.
3. **Builds payload:** Calls `_build_offer_update_payload()` (see §7).
4. **Existing draft check:** If `existing_draft` is not None, appends a
   warning to the confirmation prompt telling the user their existing draft
   will be overwritten.
5. **Returns:** A `confirmationRequired=True` response with `confirmationPrompt`
   that the LLM must show verbatim.

### Returns

```json
{
  "success": true,
  "confirmationRequired": true,
  "pendingAction": "Save draft for offer ec140104-...",
  "routedTo": "draft",
  "changes": { "offerBid": { "previous": 1.7, "new": 1.8 } },
  "confirmationPrompt": "...",
  "offerUniqueId": "ec140104-...",
  "executeSteps": ["LEADGEN", "MAX_CAMPAIGN", "MAX_ADGROUP", "MAX_ADGROUP_TARGETTING"],
  "existingDraftFound": true,
  "existingDraft": { ... }
}
```

---

## 6. Tool: `confirm_offer_update` (Phase 2 — Save Draft)

**File:** `mcp_server/server.py`

**Purpose:** Actually save the draft after user confirmation.

### When to call

Only after `update_offer` returned `confirmationRequired=True` AND the user
explicitly said "yes" or "confirm". The LLM enforces this via the tool description.

### What it does

1. **Guards:** Returns an error if no change fields were passed.
2. **Re-fetches:** Calls `_build_offer_update_payload()` fresh — same parameters
   as Phase 1. This guarantees the draft is built from the latest live data, not
   a cached snapshot from the Phase 1 call.
3. **Saves draft:** Calls `api.save_draft(settings, {"data": data})`.
4. **Builds console URL:** Calls `_build_console_url(offer_type, offer_unique_id)`.
5. **Returns:** Success response with the staging console URL.

### Returns

```json
{
  "success": true,
  "draftSaved": true,
  "offerUniqueId": "ec140104-...",
  "offerType": "hostpost",
  "changes": { "offerBid": { "previous": 1.7, "new": 1.8 } },
  "executeSteps": ["LEADGEN", "MAX_CAMPAIGN", "MAX_ADGROUP", "MAX_ADGROUP_TARGETTING"],
  "activityLog": [{ "fieldName": "offerBid", "currentValue": 1.8, "previousValue": 1.7 }],
  "message": "Draft saved successfully. Click the link below to review...",
  "consoleUrl": "https://staging-console.customer-acquisition.co/hostpost/ec140104-..."
}
```

---

## 7. Internal Helper: `_build_offer_update_payload`

**File:** `mcp_server/server.py`

This is the core of the update logic. It is called **twice** — once by
`update_offer` (preview) and once by `confirm_offer_update` (execution). By
calling it fresh both times, Phase 2 never depends on any state held from Phase 1.

### Steps inside the function

```
1. api.get_offer()
       ↓
   raw response: { response: { live: {...}, draft: {...} } }
       ↓
2. _extract_live()    → live offer dict (error if missing)
   _extract_draft()   → existing draft dict (None if none)
       ↓
3. offer_type = live["offerType"]                    # e.g. "hostpost"
   data = build_live_offer_for_draft(live)           # deep-copy, strip offerActivityLog
   before = copy.deepcopy(data)                      # snapshot BEFORE mutations
       ↓
4. Apply mutations to `data`:
   • new_offer_bid        → data["offerBid"] = new_offer_bid
   • new_offer_name       → data["offerName"] = new_offer_name
   • new_daily_budget     → data["maxData"]["campaignData"]["budget"]["dailyLimit"]
   • new_overall_budget   → data["maxData"]["campaignData"]["budget"]["overallLimit"]
   • new_offer_sub_type   → data["offerSubType"] = new_offer_sub_type
       ↓
5. ActivityLogService.get_activity_logs(before, data, exclude_fields=[...])
       → [ {fieldName, previousValue, currentValue}, ... ]
       ↓
6. Build `changes` dict for the UI:
       { fieldName: { previous: ..., new: ... } }
       ↓
7. data["activityLog"] = activity_log
   valid_steps = steps_valid_for_offer_type(offer_type)
   execute_steps = evaluate_offer_steps(live, activity_log, valid_steps)
   data["executeSteps"] = execute_steps
   data["currentVersion"] += 0.01
       ↓
8. Extract maxAdGroupId from live.maxData.adgroupData.id
   data["maxAdGroupId"] = max_adgroup_id
       ↓
9. Return { success, data, live, existing_draft, offer_type,
            changes, activity_log, execute_steps, max_adgroup_id }
```

---

## 8. Internal Helper: `_make_confirmation_response`

**File:** `mcp_server/server.py`

Builds the standardised "pending confirmation" response that every write tool
returns from Phase 1. This helper is intentionally generic so it can be reused
by any future write tool that follows the same two-phase pattern.

### What it builds

```
confirmationPrompt =
  "Confirmation required — <action>

   Pending changes:
     • offerBid: 1.7 → 1.8

   Changes will be saved as a draft. You will receive a staging
   console link to review and promote to live.

   ⚠️  An existing draft was found for this offer (bid: $5).
       Saving new changes will OVERWRITE this existing draft.

   Reply 'yes' or 'confirm' to save the draft, or 'no' / 'cancel' to abort."
```

The LLM is instructed to show this **verbatim** — it must not paraphrase it.

---

## 9. ActivityLogService — Diffing Two Snapshots

**File:** `mcp_server/activity_log.py`

This service answers: "what changed between the live offer and the mutated payload?"

### Algorithm

1. **Deep-copy** both snapshots so the originals are never mutated.
2. **Flatten** each dict to dot-keyed leaves:
   ```python
   { "maxData.campaignData.budget.dailyLimit": 5, "offerBid": 1.7, ... }
   ```
3. **Diff** the two flat dicts key-by-key:
   - Primitive values: append an entry if they differ (string↔number loose
     equality is tolerated — `"5" == 5` is not a change).
   - Arrays of primitives: compared as sets; if different, one entry is appended
     with both sides joined as a string.
   - Arrays of objects: recursed pairwise by index.
4. **Filter** out entries where both sides are falsy.

### Output shape

```python
[
  {
    "fieldName":     "offerBid",
    "currentValue":  1.8,
    "previousValue": 1.7,
  }
]
```

### Why exclude_fields is used

`_build_offer_update_payload` excludes `["currentVersion", "activityLog", "executeSteps"]`
because these are server-managed bookkeeping fields — they are always different
between the two snapshots but do not represent a user-visible change.

---

## 10. offer_filter — executeSteps Computation

**File:** `mcp_server/offer_filter.py`

The backend's `save-draft-data` endpoint requires an `executeSteps` array that
tells it which downstream integration systems need re-syncing. The Angular
frontend computes this array client-side; this module replicates that logic in Python.

### Step lists by offer type

| offer type | Valid steps |
|---|---|
| `hostpost` | LEADGEN, MAX_CAMPAIGN, MAX_CAMPAIGN_TARGETTING, MAX_ADGROUP, MAX_ADGROUP_TARGETTING, MAX_ADGROUP_FREQUENCYCAP, MAX_AD |
| `offerwall` | Same as hostpost + HASOFFERS_CREATE, HASOFFERS_SET_AFFILIATE, HASOFFERS_TEST_LINK |
| `linkout`, `custom-linkout` | Same as offerwall + HASOFFERS_GEOTARGETING |

### `evaluate_offer_steps` algorithm

```
1. Start with the offer's existing executeSteps as the initial set.
2. For each activity-log entry:
     a. Find all OFFER_FIELDS rules whose path matches the fieldName.
     b. Add each rule's affected_steps to the running set.
     c. If no rule matches → add "ALL" (unknown field, re-run everything).
3. If "ALL" is in the set → return all valid steps for this offer type.
4. Otherwise → return the steps in canonical order, filtered to the valid set.
```

### OFFER_FIELDS — field → steps mapping (selected examples)

| Field | Steps triggered |
|---|---|
| `offerBid` | LEADGEN, MAX_CAMPAIGN, MAX_ADGROUP, MAX_ADGROUP_TARGETTING, HASOFFERS_CREATE |
| `offerName` | LEADGEN, MAX_CAMPAIGN, MAX_AD, HASOFFERS_CREATE |
| `maxData.campaignData.budget` | LEADGEN, MAX_CAMPAIGN |
| `maxData.campaignTargeting` | LEADGEN, MAX_CAMPAIGN_TARGETTING |
| `offerSubType` | LEADGEN, MAX_ADGROUP, MAX_ADGROUP_TARGETTING |
| `is_ping` | LEADGEN, MAX_CAMPAIGN, MAX_CAMPAIGN_TARGETTING |

### `build_live_offer_for_draft`

Before any mutations, the live offer is deep-copied and `offerActivityLog`
(the server's full historical audit log) is stripped. The draft endpoint expects
`activityLog` (the current session's changes only) — not the entire history.

---

## 11. api.save_draft — The HTTP Call

**File:** `mcp_server/api.py`

```
POST /api/admin/offer/save-draft-data
Headers:
  Authorization: Bearer <laravel_passport_token>
  Cf-Access-Jwt-Assertion: <cloudflare_jwt>   (only if behind CF Access)
  Content-Type: application/json

Body:
{
  "data": {
    "offerUniqueId": "ec140104-...",
    "offerBid": 1.8,
    "currentVersion": 1.15,
    "activityLog": [
      { "fieldName": "offerBid", "currentValue": 1.8, "previousValue": 1.7 }
    ],
    "executeSteps": ["LEADGEN", "MAX_CAMPAIGN", "MAX_ADGROUP", "MAX_ADGROUP_TARGETTING"],
    "maxAdGroupId": 111897,
    ... (all other live offer fields)
  }
}
```

The response is returned as-is to `confirm_offer_update`. A non-2xx response
is caught by `_handle_http_error` and returned as `{"success": False, "error": "HTTP 4xx: ..."}`.

---

## 12. Console URL Construction

**File:** `mcp_server/server.py` — `_build_console_url`

### New route format (current)

```
{console_base_url}/{offertype}/{offer_unique_id}
```

`console_base_url` is set in `config.py`:
```
https://staging-console.customer-acquisition.co
```

### Slug map

Most offer types use their type name directly as the URL segment. The only
exception is `custom-linkout`, which maps to `linkout` (matching the console's
route naming):

```python
_CONSOLE_TYPE_SLUG: dict[str, str] = {
    "custom-linkout": "linkout",
}
```

| offer type | URL segment | Example URL |
|---|---|---|
| `hostpost` | `hostpost` | `.../hostpost/ec140104-...` |
| `linkout` | `linkout` | `.../linkout/abc123-...` |
| `custom-linkout` | `linkout` | `.../linkout/def456-...` |
| `offerwall` | `offerwall` | `.../offerwall/ghi789-...` |

### Previous format (removed)

Before this change, every type had a `-tp` suffix (e.g. `hostpost-tp`, `linkout-tp`).
This was hardcoded in `_CONSOLE_TYPE_SLUG` and the fallback `f"{offer_type}-tp"`.
The new format drops the suffix entirely and matches the actual console routes.

---

## 13. Auth — Who Makes the API Call?

**File:** `mcp_server/server.py` — `_get_user_creds`

The MCP server supports two auth modes:

### HTTP mode (AUTH_ENABLED=true, Cloudflare Access in front)

1. Cloudflare validates the user's browser session and forwards a `Cf-Access-Jwt-Assertion`
   JWT to the MCP server.
2. `CFAuthMiddleware` (in `auth.py`) validates the JWT and sets `current_identity`
   (a contextvar carrying email + sub).
3. `_get_user_creds()` checks the in-memory `token_store` for a cached Laravel
   Passport token keyed by the CF `sub`.
4. On a cache miss (or `force=true` login), it calls `api.laravel_login()`:
   `POST /api/auth/login` with the CF JWT → Laravel returns a Passport token.
5. The Passport token is cached until its `expires_at` and reused for all
   subsequent tool calls from the same user.
6. On any 401 response, `_invalidate_on_auth_error()` drops the cached token
   so the next call re-logs in automatically.

### stdio mode (AUTH_ENABLED=false, local dev)

`current_identity` is never set. `_get_user_creds()` returns `(None, None)`.
`api._headers()` falls back to `settings.api_token` (from `.env`).

---

## 14. Worked Example: Bid 1.7 → 1.8

This is the exact sequence that ran when you updated offer `ec140104-...`.

### Phase 1 — update_offer

```
Input:  offer_unique_id="ec140104-...", new_offer_bid=1.8

_build_offer_update_payload:
  GET /api/admin/offer/get-offer/ec140104-...
  live.offerBid = 1.7
  existing_draft.offerBid = 5  (a draft already existed from shivank)

  data = deepcopy(live), strip offerActivityLog
  before = deepcopy(data)
  data["offerBid"] = 1.8

  ActivityLogService.get_activity_logs(before, data):
    → [{ fieldName: "offerBid", previousValue: 1.7, currentValue: 1.8 }]

  evaluate_offer_steps for "hostpost":
    offerBid → hits OFFER_FIELDS rule → steps: LEADGEN, MAX_CAMPAIGN, MAX_ADGROUP, MAX_ADGROUP_TARGETTING
    → ["LEADGEN", "MAX_CAMPAIGN", "MAX_ADGROUP", "MAX_ADGROUP_TARGETTING"]

  data["currentVersion"] = 1.14 + 0.01 = 1.15

Output (to LLM):
  confirmationRequired: true
  changes: { offerBid: { previous: 1.7, new: 1.8 } }
  existingDraftFound: true   ← warns user the $5 draft will be overwritten
  confirmationPrompt: "... offerBid: 1.7 → 1.8 ... existing draft found (bid: $5) ..."
```

### Phase 2 — confirm_offer_update

```
Input: same parameters as Phase 1

_build_offer_update_payload:  (re-fetches live fresh)
  same computation as above → data with offerBid=1.8, currentVersion=1.15

api.save_draft(settings, {"data": data})
  POST /api/admin/offer/save-draft-data  → success

_build_console_url("hostpost", "ec140104-...")
  → "https://staging-console.customer-acquisition.co/hostpost/ec140104-..."

Output (to LLM):
  draftSaved: true
  consoleUrl: "https://staging-console.customer-acquisition.co/hostpost/ec140104-..."
```

---

## 15. What Is Deliberately Disabled

Two API paths exist in `api.py` but are commented out:

### `update_offer_direct` (disabled)

Would call `PUT /api/admin/offer/update-offer-complete/{type}/update` with
`saveType='direct'` — meaning the PHP backend uses the request body directly
without reading from the draft table (no human review step). Disabled because
all changes now require human review via the draft flow.

### `promote_offer_draft` (disabled)

Would call the same endpoint but without `saveType` — meaning the PHP backend
reads the draft from the `offer_third_party_integration` table and promotes it
to live. Disabled because the MCP server no longer promotes to live. Promotion
is done manually by the user in the staging console UI.

---

## 16. Data Flow Diagram

```
User (Cursor chat)
     │
     │ "update bid to 1.8"
     ▼
LLM (Claude / GPT)
     │ reads tool descriptions
     │ decides: update_offer(offer_unique_id, new_offer_bid=1.8)
     ▼
FastMCP server (server.py)
     │
     ├─► _get_user_creds()
     │       └─► token_store.get(sub)      ← cache hit?
     │               miss → api.laravel_login()
     │                         POST /api/auth/login  (CF JWT → Passport token)
     │
     ├─► _build_offer_update_payload()
     │       │
     │       ├─► api.get_offer()
     │       │       GET /api/admin/offer/get-offer/{uuid}
     │       │       ← { live: {...}, draft: {...} }
     │       │
     │       ├─► build_live_offer_for_draft(live)   [offer_filter.py]
     │       │       deep-copy + strip offerActivityLog
     │       │
     │       ├─► Apply mutations (offerBid = 1.8)
     │       │
     │       ├─► ActivityLogService.get_activity_logs(before, after)  [activity_log.py]
     │       │       flatten → diff → [{ fieldName, previousValue, currentValue }]
     │       │
     │       └─► evaluate_offer_steps(live, activity_log, valid_steps)  [offer_filter.py]
     │               OFFER_FIELDS lookup → affected steps union → canonical order
     │
     ├─► _make_confirmation_response()   [Phase 1 only]
     │       builds confirmationPrompt with existing-draft warning
     │
     └─► [after user says "yes"] api.save_draft()
             POST /api/admin/offer/save-draft-data
             ← { success: true }
             │
             └─► _build_console_url(offer_type, offer_unique_id)
                     → https://staging-console.customer-acquisition.co/{offertype}/{uuid}
                     returned to LLM → shown to user
```
