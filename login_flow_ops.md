# Login & Auth Flow — Leadgen Ops System

> Complete documentation of how authentication works across the three-tier
> Leadgen ops system: Ops UI → Cloudflare → Ops Dashboard Backend.

---

## System Architecture — Three Components

Before diving into the login flow, it is critical to understand what the
three components are and how they relate to each other.

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. OPS UI                                                          │
│     The frontend web application                                    │
│     URL: https://staging-console.customer-acquisition.co           │
│     What it is: React/Vue dashboard used by the ops team            │
│     What it does: Renders pages, lets users make changes,           │
│                   calls the Ops Dashboard Backend via HTTP          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  All API calls go through here
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. CLOUDFLARE (CF Access)                                          │
│     The security gateway — sits in FRONT of the backend            │
│     What it is: Cloudflare Zero Trust / Access                      │
│     What it does:                                                   │
│       - Blocks all unauthenticated traffic before it hits Laravel   │
│       - Handles Google OAuth2 login (the "Sign in with Google" flow)│
│       - Issues signed JWTs (CF_Authorization cookie) to browsers    │
│       - Injects Cf-Access-Jwt-Assertion header to trusted requests  │
│     The backend CANNOT be reached without passing CF first          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  Only verified requests forwarded
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  3. OPS DASHBOARD BACKEND                                           │
│     This repository — the Laravel API                               │
│     URL: https://staging-console.customer-acquisition.co/backend   │
│     What it is: Laravel PHP API (this repo)                         │
│     What it does:                                                   │
│       - Receives requests forwarded by CF                           │
│       - Verifies the CF JWT header                                  │
│       - Issues its own Passport Bearer tokens                       │
│       - Performs all business logic (offers, campaigns, leads, etc.)│
└─────────────────────────────────────────────────────────────────────┘
```

**Key point:** The Ops UI never talks to the backend directly. Every single
request goes through Cloudflare first. If CF rejects a request, Laravel
never even sees it.

---

## The Complete Login Flow

### What triggers a login

A user logs out (or their session expires) and tries to access the ops
dashboard. The Ops UI detects there is no valid `access_token` in
`localStorage` and sends the user to `/login`.

---

### Phase 1 — Google OAuth2 via Cloudflare Access

This phase happens **entirely inside Cloudflare** — no Laravel code runs.

```
User visits https://staging-console.customer-acquisition.co/login
                        │
                        ▼
          Cloudflare Access checks: does this browser
          have a valid CF_Authorization cookie?
                        │
           ┌────────────┴────────────┐
           │ NO (logged out/expired) │ YES (still valid)
           ▼                         ▼
  CF redirects user to         CF forwards request
  Google OAuth2 login          directly to Laravel
           │                   (skip to Phase 2)
           ▼
  User sees Google "Sign in"
  screen — picks their Gmail
           │
           ▼
  Google verifies credentials,
  sends OAuth2 callback to CF
           │
           ▼
  CF validates the Google token
  Checks: is this email allowed
  by the CF Access policy?
           │
    ┌──────┴──────┐
    │ Allowed     │ Not allowed
    ▼             ▼
  CF issues    CF shows 403
  JWT cookie   "Access Denied"
           │
           ▼
  Browser receives two CF cookies:
    CF_Authorization = <signed JWT>
    CF_AppSession    = <session id>
           │
           ▼
  CF redirects browser back to
  the original ops UI URL
```

**What CF_Authorization contains (decoded):**
```json
{
  "aud": ["f5a60af8b11d737cd11e73067a3bf5c013929511bef5eefa1038fa4065ef440e"],
  "email": "arihant@customer-acquisition.co",
  "exp": 1776488551,
  "iat": 1776229351,
  "iss": "https://custac.cloudflareaccess.com",
  "type": "app",
  "sub": "31169eaf-f4a0-4af3-ad69-4ee80f807da1",
  "country": "IN",
  "policy_id": "773fd9ef-40f6-4e58-949b-1e95ce83ccf8"
}
```

| Field | Meaning |
|---|---|
| `email` | The Google Gmail address the user logged in with |
| `exp` | Token expiry (~72 hours from issue) |
| `aud` | Your CF application ID — CF validates this |
| `iss` | CF team domain — proves token came from your CF account |
| `sub` | CF's internal UUID for this user |
| `policy_id` | Which CF access policy allowed this user |

---

### Phase 2 — Ops UI calls the Backend Login Endpoint

Now the browser has a valid `CF_Authorization` cookie. The Ops UI
JavaScript calls the backend to exchange the CF identity for a Laravel
Passport token.

```
POST https://staging-console.customer-acquisition.co/backend/api/auth/login

Cookies sent automatically by browser:
  CF_Authorization = <CF JWT>
  CF_AppSession    = <CF session id>
  XSRF-TOKEN       = <Laravel CSRF token>
  laravel_session  = <Laravel session id>

Headers sent by Ops UI:
  authorization: Bearer          ← empty, no token yet
  x-xsrf-token: <csrf value>    ← CSRF protection
  content-length: 0              ← no body, no password
  origin: https://staging-console...
  referer: .../login?redirectURL=/lead/listing
```

**Why `content-length: 0` and no password:**
The Ops UI sends an empty POST body. No email, no password. CF already
proved who the user is via Google OAuth2. The backend will read the
identity from the CF JWT, not from a password.

---

### Phase 3 — Cloudflare Forwards the Request to Laravel

CF intercepts the request before it reaches Laravel and does two things:

```
Browser request arrives at CF edge
              │
              ▼
CF checks CF_Authorization cookie:
  - Signature valid? (RS256, signed by CF private key)
  - Not expired? (exp > now)
  - Audience matches? (aud = your CF app ID)
  - User's email allowed by policy?
              │
    ┌─────────┴──────────┐
    │ PASS               │ FAIL
    ▼                    ▼
CF injects header:   CF blocks request
Cf-Access-Jwt-Assertion: <same JWT value>
              │
              ▼
CF forwards request to Laravel origin server
(the backend — which is NOT publicly accessible)
```

**Critical distinction:**
- `CF_Authorization` is a **cookie** — set in the browser, sent by the browser
- `Cf-Access-Jwt-Assertion` is a **header** — injected by CF's edge servers

Laravel only reads the **header**, never the cookie directly. This matters
because the header can ONLY be injected by CF (the browser cannot set
arbitrary headers like this on cross-origin requests). It is proof the
request was vetted by CF.

---

### Phase 4 — Laravel Verifies the CF JWT and Issues a Passport Token

`AuthController::login()` runs:

**Step 1 — Read the injected header**
```php
$cloudFlareToken = $request->header('Cf-Access-Jwt-Assertion');
```
If this header is present, we are in the CF path (staging/production).
If absent, Laravel falls back to email+password (local dev only).

---

**Step 2 — Parse the JWT header to get the key ID**
```php
$tokenDetails = explode('.', $cloudFlareToken);
$header = (array)json_decode(base64_decode($tokenDetails[0]));
// $header['kid'] = "7cadf3c9bd4beedbfe6f0c0064e855ccfd5f11af..."
```
A JWT has three base64-encoded parts: `header.payload.signature`.
The header contains `kid` — the ID of the key that signed this token.

---

**Step 3 — Fetch CF's public keys**
```php
$data = Http::get($request->getSchemeAndHttpHost().'/cdn-cgi/access/certs')->json();
```
Calls: `https://staging-console.customer-acquisition.co/cdn-cgi/access/certs`

This is a CF-served JWKS (JSON Web Key Set) endpoint. It returns all
current public keys CF uses to sign JWTs:
```json
{
  "public_certs": [
    {
      "kid": "7cadf3c9bd4beedbfe6f0c0064e855ccfd5f11af...",
      "cert": "-----BEGIN CERTIFICATE-----\nMIIC..."
    }
  ]
}
```

---

**Step 4 — Match the correct public key**
```php
$publicKey = collect($data['public_certs'])->first(function($keyData) use ($header) {
    return $keyData['kid'] === $header['kid'];
});
```
Finds the cert whose `kid` matches the JWT's `kid`. CF rotates keys
periodically — this lookup ensures the right key is always used.

---

**Step 5 — Cryptographically verify the JWT**
```php
$decoded = JWT::decode($cloudFlareToken, new Key($publicKey['cert'], 'RS256'));
```
Uses `firebase/php-jwt` to verify:
- The signature is valid (only CF's private key could have produced it)
- The token is not expired (`exp > now`)
- The algorithm is RS256

If anything fails:
```json
{ "message": "Your Cloudflare JWT token is invalid", "status": 400 }
```

---

**Step 6 — Extract email and look up the user**
```php
$email = $decoded->email;
$user = AdminUsers::where('email', $email)->first();
```
The email comes from the Google account the user signed in with.
It must exist in the `admin_users` table in the backend database.

If the user is not found:
```json
{ "message": "User arihant@customer-acquisition.co does not exist in the system", "status": 400 }
```

This is why onboarding a new team member requires TWO things:
1. Add their Gmail to the **CF Access policy** (so CF lets them through)
2. Create a row in **`admin_users`** with their email + role (so Laravel recognises them)

---

**Step 7 — Create a Laravel Passport token**
```php
$tokenResult = $user->createToken('Personal Access Token');
$token = $tokenResult->token;
$token->save();
```
Laravel Passport generates a new OAuth2 Bearer token and stores it in the
`oauth_access_tokens` table. Expiry variations:
- Default: standard Passport expiry
- `remember_me: true` in request → expires in 1 week
- `persistent: true` + Super Admin → expires in 5 years

---

**Step 8 — Return the Passport token to the Ops UI**
```json
{
  "client_id": "1",
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9...",
  "userName": "arihant@customer-acquisition.co",
  "userInitial": "Arihant",
  "token_type": "Bearer",
  "expires_at": "2026-04-17 10:05:00",
  "maxRedirectUrl": "https://...",
  "maxAccessStatus": "..."
}
```

The Ops UI stores `access_token` in `localStorage` and uses it as
`Authorization: Bearer <token>` on every subsequent API call.

---

### Phase 5 — All Subsequent API Calls (After Login)

```
Ops UI
  │  GET /backend/api/admin/offer/get-offer/{uuid}
  │  Authorization: Bearer eyJ0eXAiOiJKV1Qi...   ← Passport token
  │  Cookie: CF_Authorization=<CF JWT>            ← CF still validates
  ▼
Cloudflare Edge
  │  Validates CF_Authorization cookie (still active)
  │  Forwards + injects Cf-Access-Jwt-Assertion header
  ▼
Laravel
  │  auth:api middleware (if enabled on this route)
  │    → Passport checks the Bearer token in oauth_access_tokens
  │    → Verifies not revoked, not expired
  │  roles middleware (if enabled)
  │    → Checks user's role against allowed list
  ▼
Controller runs — returns data
```

Two independent auth layers are active on every call:
- **CF layer**: verifies the CF JWT (edge-level)
- **Laravel layer**: verifies the Passport Bearer token (app-level, on protected routes)

---

## Why Login is Instant After Logout

```
User clicks Logout
      │
      ▼
Laravel: $request->user()->token()->revoke()
  → Sets revoked = 1 in oauth_access_tokens
  → The Passport token is now invalid

CF_Authorization cookie is NOT cleared
  → CF session is still active (72h lifetime)

User hits /login again
      │
      ▼
Ops UI calls POST /backend/api/auth/login
      │
      ▼
CF checks CF_Authorization cookie → still valid → forwards immediately
      │
      ▼
Laravel reads Cf-Access-Jwt-Assertion → verifies → finds user → creates NEW Passport token
      │
      ▼
Ops UI has a new access_token — user is back in, no Google screen shown
```

This is intentional. Logging out of the ops dashboard only invalidates the
Laravel Passport token. The CF/Google session remains active until:
- The `CF_Authorization` JWT expires (~72 hours), OR
- The user explicitly signs out of CF:
  ```
  https://custac.cloudflareaccess.com/cdn-cgi/access/logout
  ```

---

## All Cookies and Headers — Reference Table

### Cookies (set in browser, sent automatically)

| Cookie | Set by | Lifetime | Purpose |
|---|---|---|---|
| `CF_Authorization` | Cloudflare Access (after Google OAuth2) | ~72 hours | Signed JWT proving Google identity. CF validates this on every request. |
| `CF_AppSession` | Cloudflare Access | ~72 hours | CF edge session tracker — lets CF skip full JWT re-check on repeat requests |
| `XSRF-TOKEN` | Laravel (on first page load) | Per session | Encrypted CSRF token. Frontend reads and mirrors in `x-xsrf-token` header |
| `laravel_session` | Laravel (on first page load) | Per session | Laravel server-side session ID (maps to session data on server) |

### Headers (set by Ops UI JavaScript)

| Header | Value in login call | Purpose |
|---|---|---|
| `authorization` | `Bearer` (empty) | No token yet — this is the call to GET the token |
| `x-xsrf-token` | CSRF token value | Mirrors `XSRF-TOKEN` cookie — CSRF protection |
| `content-length` | `0` | Empty body — no password sent |
| `origin` | `https://staging-console...` | CORS — Laravel/CF verify this matches allowed origins |
| `referer` | `.../login?redirectURL=...` | Frontend uses `redirectURL` to send user back after login |

### Header injected by Cloudflare (never from browser)

| Header | Set by | Purpose |
|---|---|---|
| `Cf-Access-Jwt-Assertion` | CF edge servers only | The CF JWT forwarded to Laravel for verification. Browsers cannot set this — it is proof of CF vetting |

---

## Two Login Paths in the Code

```php
// AuthController::login()

if (isset($cloudFlareToken)) {
    // ── Path A: Staging / Production ──────────────────────────────
    // CF is in front. No password needed.
    // Identity proven by Google OAuth2 → CF JWT → Cf-Access-Jwt-Assertion header.
    $decoded = JWT::decode($cloudFlareToken, ...);
    $user = AdminUsers::where('email', $decoded->email)->first();

} else {
    // ── Path B: Local Development ──────────────────────────────────
    // No CF in front. Standard email + password login.
    $credentials = request(['email', 'password']);
    if (!Auth::attempt($credentials)) {
        return response()->json(['message' => 'Unauthorized'], 401);
    }
    $user = $request->user();
}

// Both paths end here — same Passport token issued
$tokenResult = $user->createToken('Personal Access Token');
```

| | Path A — CF (Staging/Prod) | Path B — Email+Password (Local) |
|---|---|---|
| **Trigger** | `Cf-Access-Jwt-Assertion` header present | Header absent |
| **Identity source** | Google OAuth2 via Cloudflare | bcrypt password in `admin_users` |
| **Body required** | Empty (`content-length: 0`) | `email` + `password` |
| **Where used** | Staging, Production | Local `php artisan serve` |

---

## Onboarding a New Team Member — Checklist

Because auth is split across CF and Laravel, two separate steps are required:

- [ ] **Step 1 — Cloudflare Access**
  Add the person's `@customer-acquisition.co` Gmail to the CF Access policy
  `773fd9ef-40f6-4e58-949b-1e95ce83ccf8` (or allow the whole domain).
  Without this, CF blocks them before they reach Laravel.

- [ ] **Step 2 — admin_users table**
  Insert a row in the `admin_users` table with their `email` and the
  appropriate `role_id`.
  Without this, CF lets them through but Laravel returns:
  `"User <email> does not exist in the system"`

---

## Common Errors and Their Causes

| Error | HTTP | Where it's thrown | Cause |
|---|---|---|---|
| CF login page / 403 | 403 | Cloudflare | User's Gmail not in CF Access policy |
| `"Your Cloudflare JWT token is invalid"` | 400 | Laravel `AuthController` | CF JWT expired, wrong audience, or signature mismatch |
| `"User <email> does not exist in the system"` | 400 | Laravel `AuthController` | Passed CF but not in `admin_users` table |
| `"Unauthorized"` | 401 | Laravel `AuthController` | Path B only — wrong email or password |
| `419 Page Expired` | 419 | Laravel CSRF middleware | `x-xsrf-token` missing or doesn't match `XSRF-TOKEN` cookie |
| `"Unauthenticated."` | 401 | Laravel `auth:api` middleware | Calling a protected route without a valid Passport Bearer token |
| `"Your role X is not authorized"` | 403 | Laravel `CheckRole` middleware | Valid token but user's role not in the allowed list for that route |
| `Too Many Requests` | 429 | Laravel throttle middleware | Exceeded 256 requests/minute from same IP |
