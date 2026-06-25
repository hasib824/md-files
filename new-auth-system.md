# Auth System — Overview

> **30-second summary:** When you log in, the server gives your browser a special **cookie** that
> holds your login token. The browser sends it automatically on every request. The token can't be
> read by JavaScript, can't be read by other websites, and each device's login can be logged out on
> its own. A second cookie protects against fake requests from other sites. The **API gateway** is the
> guard that checks all of this on every request.

---

## A few words first (mini-glossary)

- **JWT (token):** a signed string that proves "this is user X." The server signs it so it can't be
  faked. Anyone *holding* it can use it (like a movie ticket), so we must keep it safe.
- **Cookie:** a small piece of data the server asks the browser to store and **send back
  automatically** on every request to that server.
- **HttpOnly cookie:** a cookie that **JavaScript cannot read.** Only the browser handles it. This is
  the key safety feature.
- **SameSite:** a cookie setting that controls whether the cookie is sent when the request comes from
  *another* website.
- **Origin vs Site:** an *origin* is the exact `https://host` (e.g. `app.cptos.com`). A *site* is the
  registrable domain (e.g. `cptos.com`). `app.cptos.com` and `api.cptos.com` are different origins but
  the **same site**.
- **XSS:** an attack where the attacker manages to run **their JavaScript inside our own page.**
- **CSRF:** an attack where **another website** secretly makes your browser send a request to our API,
  riding on your logged-in cookie.

---

## Part 1 — The old system, and what was wrong with it

**How it used to work:**
- After login, the browser saved the JWT in **`localStorage`** (a JavaScript storage box).
- On each request, the frontend manually attached it as a header: `Authorization: Bearer <token>`.
- Logout increased a single counter on the user's database row called **`token_version`**. Every
  token carried that number; the gateway rejected any token whose number didn't match.

**Problem 1 — logging out one device logged out ALL devices.**
`token_version` was **one number for the whole user.** If you were logged in on your office PC and your
laptop, clicking "logout" anywhere increased the number → *every* device's token became invalid at
once. There was no way to tell "office PC's token" apart from "laptop's token."

**Problem 2 — the token could be stolen by a script.**
Because the token lived in `localStorage`, **any JavaScript on the page could read it** (`localStorage.getItem(...)`).
If an attacker ever injected a script (XSS), they could copy the token and use your account from
anywhere. The token was sitting in the open.

---

## Part 2 — What we use now

We fixed both problems and hardened the whole flow. Four ideas:

**1. The token now lives in an HttpOnly cookie (`access_token`).**
The server sets it at login; the browser sends it automatically. Because it's **HttpOnly**, no
JavaScript can read it — not ours, not an attacker's. This solves Problem 2: even if a bad script runs
on the page, it cannot read or steal the login token.

**2. Each login has its own session id (`sid`), so logout is per-device.**
Every time you log in, the server creates a unique **session id** and puts it inside the token, and
records that session in the `login_tracking` table. Logout closes **only that one session.** So
logging out on your laptop leaves your office PC logged in. (The old `token_version` counter still
exists, but now it's only the **"log this user out of ALL devices"** lever — used when an admin
suspends an account, etc.)

**3. The API gateway is the single guard.**
Every request goes through the gateway first. It reads the token from the cookie and checks:
signature & expiry, the session is still open (`sid`), the user is still ACTIVE, and the
`token_version` matches. Only then does it pass the request to the real service.

**4. A second cookie protects against fake cross-site requests (CSRF).**
The gateway also gives the browser a **readable** cookie called `XSRF-TOKEN`. Our frontend (axios)
reads it and copies it into a header `X-XSRF-TOKEN` on every change-making request. The gateway checks
that the cookie and the header match. A different website can't read your `XSRF-TOKEN`, so it can't
fake this — that's the whole trick (called "double-submit").

### Before → after

| | Old | Now |
|---|---|---|
| Where the token lives | `localStorage` (JS can read) | **HttpOnly cookie** (JS cannot read) |
| How it's sent | manual `Authorization: Bearer` header | browser sends the cookie automatically |
| Logout scope | all devices at once | **only the device that logs out** |
| "Log out everywhere" | (the only option) | still possible via `token_version` (admin action) |
| Fake-request (CSRF) defense | not needed (header) | `SameSite` + `XSRF-TOKEN` double-submit |

---

## Part 3 — What can be seen, and by whom

This is the part that confuses people. Seeing a cookie in your own browser is **not** a security hole.
The protection is about *who else* can read it.

**You, looking at your own browser's DevTools:** you see **both** cookies (`access_token` and
`XSRF-TOKEN`). That's completely normal — it's *your* session on *your* screen. You could always see
your own session.

**Our app's own JavaScript (running on cptos.com):**
- can read `XSRF-TOKEN` (it must, to send the header), but
- **cannot** read `access_token` — it's HttpOnly. Open the browser console and type `document.cookie`:
  you'll see `XSRF-TOKEN=...` but **not** `access_token`. That's the protection working.

**Another website (the attacker's `evil.com`):**
- **Cannot read either cookie. Ever.** A browser keeps each site's cookies in a separate box, and a
  website can only read its **own** cookies. `evil.com` has no way to read `cptos.com`'s cookies. This
  is built into every browser and is always on.

| Who is looking | `access_token` | `XSRF-TOKEN` |
|---|---|---|
| You, in your own DevTools | visible (your session) | visible (your session) |
| Our app's JS (on cptos.com) | ❌ cannot read (HttpOnly) | ✅ can read (by design) |
| **Another site (`evil.com`)** | ❌ **cannot read** | ❌ **cannot read** |

So: the user seeing both cookies is expected and safe. The dangerous readers — random scripts and
other sites — are locked out.

---

## Part 4 — How a request travels

### 4.1 Login
```
Browser ──POST /api/auth-service/login (username+password)──►  Gateway ──►  auth-service
                                                                                 │ checks password,
                                                                                 │ creates sid,
                                                                                 │ makes JWT
Browser  ◄──── Set-Cookie: access_token=<JWT>  (HttpOnly) ──────────────────────┘
         ◄──── Set-Cookie: XSRF-TOKEN=<random> (readable) ◄── Gateway
         ◄──── body: { roles, permissions, menu }   (NO token in the body)
```
The browser now silently holds two cookies. The frontend stores only the non-secret `roles` /
`permissions` / `menu` to draw the screen.

### 4.2 A normal read request (GET)
```
Browser ──GET /api/.../something──►  Gateway
   (browser auto-attaches              │ reads token from access_token cookie
    the access_token cookie)           │ valid signature? not expired? sid still open?
                                       │ user ACTIVE? token_version matches?
                                       ▼ all yes
                                    forwards to the service ──► 200 OK
```
You don't write any code to attach the token — the browser does it because it's a cookie.

### 4.3 A change request (POST/PUT/PATCH/DELETE)
```
Browser ──POST /api/.../create──►  Gateway
   cookie: access_token              │ 1) CSRF check: does X-XSRF-TOKEN header
   cookie: XSRF-TOKEN                 │    match the XSRF-TOKEN cookie?   ── no ──► 403
   header: X-XSRF-TOKEN  ◄─ axios     │ 2) token checks (same as GET)     ── no ──► 401
   copies it from the cookie          ▼ all yes
                                    forwards to the service ──► 200/201
```
axios adds the `X-XSRF-TOKEN` header automatically because it can read the `XSRF-TOKEN` cookie.

### 4.4 Logout
```
Browser ──POST /api/auth-service/logout──►  Gateway ──►  auth-service
   cookie: access_token                       │             │ closes THIS session (sid) only
   header: X-XSRF-TOKEN                        │             │ (other devices stay logged in)
Browser ◄── Set-Cookie: access_token=; Max-Age=0 ──────────┘  (browser deletes the cookie)
```

---

## Part 5 — What gets blocked, and exactly where

This is the "where does a request fail?" map. Almost everything is decided at the **gateway**.

| Situation | Result | Where / why |
|---|---|---|
| No `access_token` cookie (not logged in) | **401** | Gateway: no token to validate |
| Token signature wrong or **expired** | **401** | Gateway: JWT invalid |
| Token has no `sid` | **401** | Gateway: fail-closed |
| That session was **logged out** (its `sid` row closed) | **401** | Gateway asks auth-service; session not open |
| Admin did "log out everywhere" (`token_version` bumped) | **401** | Gateway: version mismatch |
| Account no longer **ACTIVE** (suspended) | **401** | auth-service `checkToken` says invalid |
| Change request **without** a correct `X-XSRF-TOKEN` | **403** | Gateway CSRF check (only `/login` is exempt) |
| Request comes from **another website** (`evil.com`) | cookie **not even sent** → **401** | `SameSite=Lax` keeps the cookie off cross-site requests |
| Another site tries to **read our API's response** in JS | browser blocks it | CORS: only our allow-listed origins may read responses |

Key takeaway: a request that is missing the cookie, carries an expired/revoked token, or lacks the
CSRF header **never reaches the business service** — it stops at the gateway with a 401 or 403.

---

## Part 6 — "Can an attacker…?" (quick answers)

**"Can `evil.com` read my login token?"**
No. A website can only read its own cookies; `evil.com` cannot read `cptos.com`'s cookies at all. And
`access_token` is HttpOnly, so not even our own scripts can read it.

**"Can `evil.com` make a request using my cookie (CSRF)?"**
Blocked twice. `SameSite=Lax` means the browser won't attach your cookie to a request triggered by
`evil.com`. And even if a request reached the API, `evil.com` can't read your `XSRF-TOKEN` to produce
the matching header → the gateway returns 403.

**"What if a bad script gets injected into OUR site (XSS)?"**
It still **cannot read `access_token`** (HttpOnly), so it can't steal the login token. XSS is always
serious, but this design removes the biggest prize. (That's also why we never put secrets in
`localStorage` anymore.)

**"Can someone copy my token to another PC and use it?"**
Any bearer token is copyable by whoever holds it — that's true of all such systems. We reduce the risk
with: HttpOnly (scripts can't grab it), HTTPS (can't be sniffed on the network), a short token
lifetime, and the ability to revoke a single session (`sid`) or all sessions (`token_version`). See
`per-device-logout.md`.

---

## Part 7 — Two end-to-end walkthroughs

**A) One user, two devices, logs out of one.**
1. Logs in on office PC → session **A** (sid A). Logs in on laptop → session **B** (sid B). Two open
   rows in `login_tracking`.
2. Clicks logout on the **laptop** → only session **B** is closed; the laptop's cookie is cleared.
3. Office PC keeps working — its token carries sid **A**, which is still open. ✅ (Old system would
   have logged out both.)

**B) `evil.com` tries to delete data using your login.**
1. You're logged into CPTOS in one tab. You visit `evil.com` in another.
2. `evil.com` runs JS that POSTs to `https://api.cptos.com/.../delete`.
3. **Checkpoint 1 (SameSite):** the browser does **not** attach your `access_token` (cross-site
   request) → the gateway sees no login → **401.** The attack dies here.
4. Even if it somehow didn't: **Checkpoint 2 (CSRF):** `evil.com` can't read your `XSRF-TOKEN`, so no
   valid `X-XSRF-TOKEN` header → **403.**
5. And `evil.com` could never read the response anyway (CORS). Nothing leaks.

---

## Quick reference

- **Cookies:** `access_token` (HttpOnly, the JWT) and `XSRF-TOKEN` (readable, CSRF token).
- **CSRF header:** `X-XSRF-TOKEN` (axios sends it automatically on change requests).
- **The guard:** the API gateway validates every request before it reaches a service.
- **Per-device logout:** `sid` per login. **Logout everywhere:** `token_version`.
- **Failure codes:** not-authenticated / bad token → **401**; failed CSRF check → **403**.

**See also:** `per-device-logout.md` (session model) · `frontend-auth-httponly-cookie-contract.md`
(frontend ⇄ backend contract) · `auth-cookie-domain-deployment.md` (deploying across domains).
