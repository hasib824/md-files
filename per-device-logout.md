# Per-Device Logout — Per-Session Token Invalidation

## 1. The Problem

Right now, when a user is logged in on two machines (say PC-A and PC-B) and logs out from **one**
of them, **both** machines get logged out. The user is forced to log in again everywhere even though
they only clicked logout on a single device.

This is a direct consequence of how token invalidation currently works, not a random bug.

## 2. Why It Happens (Root Cause)

Token revocation is keyed on a **per-user** counter called `token_version`, stored on the
`app_user` table.

The flow today:

1. **Login** — `AuthController.handleLogin` mints a JWT and copies the user's current
   `token_version` into the token as a claim called `tokenVersion`
   (`JwtUtil.generateAccessToken`).
2. **Every request** — the API gateway (`JwtAuthFilter`) reads `tokenVersion` from the JWT and asks
   auth-service (`InternalAuthController.checkToken`) whether it still matches the value in the
   database. If they match, the token is accepted; if not, it is rejected with `401`.
3. **Logout** — `AuthService.logout` calls `incrementTokenVersion(userId)`, which does
   `token_version = token_version + 1` on that user's row.

Because `token_version` is a **single number shared by the whole user**, incrementing it on logout
invalidates **every** token that user ever received — PC-A's token *and* PC-B's token both stop
matching. There is nothing in the system that distinguishes "the token from PC-A" from "the token
from PC-B". The gateway's cache key is also `userId:tokenVersion`, so the user-scoping goes all the
way through the stack.

**In one line:** one shared counter for the whole user → no way to revoke just one device.

## 3. The Fix (Industry-Standard Approach)

We introduce a **per-login session identity** so each device's token can be revoked on its own,
while keeping the existing "kill everything" capability intact.

This is the standard **stateful-JWT + server-side session allow-list** pattern. It fits this codebase
naturally because the gateway already makes a per-request validation call to auth-service
(`/check-token`) with a short cache — we are extending that existing check, not adding a new round
trip.

### Two independent revocation levers

After this change there are **two separate switches**, each answering a different question on every
request:

| Lever | Question it answers | Scope | When it fires |
|-------|---------------------|-------|---------------|
| **`session_id` (new)** | "Is *this specific device's* session still active?" | One device | Normal logout |
| **`token_version` (existing)** | "Has this *whole user* been force-logged-out everywhere?" | All devices | Admin deactivate / suspend (and future password-change, "logout everywhere") |

They are independent. Normal logout touches only the first. Admin actions touch only the second.

### What is a `session_id`?

- A random UUID the **server** generates at login. One per login event (so PC-A and PC-B get
  different ones).
- It is embedded **inside the JWT** as a claim called `sid` and is signed, so it cannot be tampered
  with.
- **The frontend does nothing new.** The React app stores and sends the JWT exactly as it does
  today (`Authorization: Bearer <token>`). The `sid` simply rides inside that token. No new field,
  no new header from the client, no permission needed.

### Validity rules after the change

A token is accepted at the gateway only if **all** of these hold:

1. Signature / issuer / audience / expiry are valid. *(unchanged)*
2. The JWT's `tokenVersion` equals `app_user.token_version`. → **global "kill all" lever.**
3. The user's status is `ACTIVE`. *(unchanged)*
4. **(new)** A `login_tracking` row exists for this user with this `session_id` **and**
   `logout_time IS NULL` (i.e. the session is still open). → **per-device lever.**

Logging out one device closes only that device's row (rule 4 fails for that token only). The other
device's token still has an open row → stays valid.

## 4. Explicit: When Does `token_version` Change?

This is the part to be crystal clear about, because the whole bug is about *when* it gets bumped.

**Today** `incrementTokenVersion` is called in two places:

- `AuthService.logout` — **on every normal logout.** ← *this is the bug; we are removing it here.*
- `AppUserService.updateByAdmin` (lines 247–250) — when an admin changes a user to a non-ACTIVE
  status (SUSPENDED, etc.). This correctly boots all of that user's sessions.

**After this change** `token_version` is bumped **only** for genuine "log this user out of
everything" events:

- ✅ **Admin deactivates / suspends a user** (`updateByAdmin`) — keeps working exactly as now.
- ✅ **(future, free) Password change** — bump it to force re-login on all devices.
- ✅ **(future, free) "Logout everywhere" button** — bump it (or bulk-close sessions).
- ❌ **Normal single-device logout** — does **NOT** touch `token_version` anymore. It only closes
  that one session row.

So: `token_version` stops being a per-logout counter and becomes purely the "kill-all" switch.
`session_id` takes over the everyday per-device logout job.

## 5. Design Decisions (agreed)

- **Session store:** reuse the existing `login_tracking` table as the session registry. It already
  has one row per login with `login_time` / `logout_time`, so an **open session = `logout_time IS
  NULL`**. We add one column for the session id. No new entity.
- **No device metadata for now:** we will **not** capture IP / user-agent yet. (They can be added
  later as columns — the server already receives them as request headers, so it would be zero
  frontend work — but they are out of scope for this change.)

## 6. Database Migration (manual — required first)

auth-service runs with `spring.jpa.hibernate.ddl-auto: validate` (`application.yaml:17`), which means
Hibernate **validates** the schema but does **not** create or alter columns. There is no
Flyway/Liquibase in the project. Therefore the new column must be added to SQL Server **by hand
before deploying**, or the service will fail to start (validation mismatch).

Run:

```sql
ALTER TABLE dbo.login_tracking ADD session_id VARCHAR(64) NULL;
CREATE INDEX IX_login_tracking_session_id ON dbo.login_tracking(session_id);
```

- `NULL`-able so existing historical rows remain valid.
- Indexed because every protected request will look a session up by `session_id`.

## 7. Backward Compatibility

JWTs already issued before deployment have no `sid` claim. Under the new rules they fail validation
(no matching open session) → those users simply log in again. Since access tokens are short-lived
(~15 minutes), this self-heals quickly and affects users only once. The gateway treats a
missing/blank `sid` as invalid (fail-closed), which is the safe default for a security change.

## 8. Code Changes — auth-service

**`model/LoginTracking.java`** — add the session id field:
```java
@Column(name = "session_id", length = 64)
private String sessionId;
```

**`repository/LoginTrackingRepository.java`** — add finders the new logic needs:
- `Optional<LoginTracking> findByAppUserIdAndSessionIdAndLogoutTimeIsNull(Long userId, String sessionId)`
  — to close the right session on logout.
- `boolean existsByAppUserIdAndSessionIdAndLogoutTimeIsNull(Long userId, String sessionId)`
  — for the gateway's per-request validity check.
- `long countByAppUserIdAndLogoutTimeIsNull(Long userId)`
  — to decide whether `app_user.is_logged_in` should become false after a single-device logout.

**`security/JwtUtil.java`** — `generateAccessToken(...)` takes a new `String sessionId` parameter and
adds `claims.put("sid", sessionId);` alongside the existing claims.

**`controller/AuthController.java`**
- `handleLogin`: generate `String sid = UUID.randomUUID().toString();`, pass it into
  `jwtUtil.generateAccessToken(...)` and into `authService.recordLogin(user.getId(), sid)`.
- `handleLogout`: read the session id from the `X-Session-Id` request header (forwarded by the
  gateway — see §9) by injecting `HttpServletRequest`, then call
  `authService.logout(userId, sessionId)`. (Reading the header directly here keeps
  `UserContextFilter` untouched.)

**`service/AuthService.java`**
- `recordLogin(Long userId, String sessionId)` → forward `sessionId` to
  `loginTrackingService.createLoginSession(user, sessionId)`. Keep `markLoggedIn`.
- `logout(Long userId, String sessionId)` — **replace the body**:
  - `loginTrackingService.closeSession(userId, sessionId)` — close only the matching open row.
  - **Remove** the `incrementTokenVersion(userId)` call. *(This single deletion is the core of the
    fix.)*
  - Set `is_logged_in = false` **only if** no other open session remains
    (`countByAppUserIdAndLogoutTimeIsNull == 0`); otherwise leave it `true` (the user is still on
    another device).

**`service/LoginTrackingService.java`**
- `createLoginSession(AppUser user, String sessionId)` — store `sessionId` on the new row.
- Add `closeSession(Long userId, String sessionId)` — find the open row by `(userId, sessionId)` and
  set its `logout_time`. If no such row exists, do nothing (do **not** fall back to closing the
  latest session — that would close the wrong device). Remove the old `closeLoginSession` if nothing
  else references it.

**`controller/InternalAuthController.java`** — `checkToken` gains a `@RequestParam String sessionId`.
After the existing user / `token_version` / `ACTIVE` checks, also require
`existsByAppUserIdAndSessionIdAndLogoutTimeIsNull(user.getId(), sessionId)`. If it is false, return
`{"valid": false}`.

**`service/AppUserService.java`** — `updateByAdmin` is **left as-is**: a status downgrade still calls
`incrementTokenVersion` + `markLoggedOut`, which (correctly) kills all of that user's devices. This is
exactly the "global lever" behavior we want to keep.

## 9. Code Changes — api-gateway

**`security/JwtAuthFilter.java`**
- Extract the new claim: `String sid = claims.get("sid", String.class);` (next to `userId` and
  `tokenVersion`). Treat null/blank as invalid → respond `unauthorized(...)` (fail-closed).
- **Cache key** changes from `userId + ":" + tokenVersion` to
  `userId + ":" + tokenVersion + ":" + sid` in **every** place it is built today
  (`validateTokenVersion`, `forwardWithHeaders`, and the logout-eviction block). This makes each
  session cache independently, so revoking one device never drops another device's cached validity.
- `validateTokenVersion(...)` passes the session id to the internal call:
  `/api/internal/check-token?username={username}&tokenVersion={tokenVersion}&sessionId={sessionId}`.
- In `forwardWithHeaders`, forward the session id downstream: `h.set("X-Session-Id", sid);` so the
  auth-service logout handler can read which device is logging out.
- The logout-eviction block (`path.equals(".../logout")`) uses the new 3-part cache key.

## 10. Verification

1. **Migration / startup:** run the `ALTER TABLE` from §6, then start auth-service. It must boot
   cleanly (ddl-auto `validate` passes). Use the isolated port-8001 instance for safe local
   verification, or the running stack.

2. **Two-device, per-device logout (the core scenario):**
   - Log in as user X → token A (sid A). Log in again → token B (sid B). DB: two `login_tracking`
     rows for X, both `logout_time NULL`, different `session_id`.
   - Hit a protected endpoint with token A → `200`; with token B → `200`.
   - `POST /logout` using **token A**.
   - Re-test: token A → `401` ("invalidated"); **token B → still `200`.** ✅
   - DB: row A has `logout_time` set; row B still open. `app_user.is_logged_in` still `true`.
   - Now logout with token B → token B → `401`; `is_logged_in` becomes `false`; both rows closed.

3. **Global kill still works:** admin sets user X to SUSPENDED via `updateByAdmin` → `token_version`
   bumps → **both** token A and token B → `401` immediately. ✅ (Proves the global lever is intact.)

4. **Cache independence:** confirm that logging out one device does not invalidate the other within
   the ~30s gateway cache window — guaranteed by the per-`sid` cache key.

5. **Legacy token:** a token minted before deployment (no `sid`) → `401`; user logs in again once.

## 11. Out of Scope (notes for later)

- **"Logout everywhere" endpoint** — implement by bumping `token_version` (or bulk-closing all open
  sessions). Trivial follow-up using the global lever.
- **Active-devices screen / revoke-by-device** — add `ip` and `user_agent` columns to
  `login_tracking` (server already receives both as headers; still zero frontend work). Deferred per
  decision in §5.
- **Refresh-token rotation** — still not implemented in the project; unchanged by this work.
