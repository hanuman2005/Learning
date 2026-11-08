# Section C — Authentication & Authorization

## C1 — AuthN vs AuthZ

**Diagram**
```
 request -> AuthN: who are you?   -- no --> 401
                 | yes
                 v
            AuthZ: role/scope ok? -- no --> 403
                 | yes
                 v
            Ownership: is it YOURS? -- no --> 403/404 (skipped = IDOR)
                 | yes
                 v
              handler
```
**In plain words.** Hotel: the front desk checks your ID (401), your key card opens only your
floor (403), and your key opens only your room (ownership). Skipping the last check lets guests
walk into each other's rooms.

- **Authentication (AuthN)** = *who are you?* Failure → **401**. Done **first**.
- **Authorization (AuthZ)** = *what may you do?* Failure → **403**.
- Menus: AuthN = password / OTP / OAuth. AuthZ = RBAC / ABAC / scopes + **ownership**.

> **One line:** AuthN = who (401), AuthZ = what (403); authenticate first.

---

## C2 — Password hashing
**Core idea.** **Hash, never encrypt** passwords (you never need to read them back).

- **Unique salt per password** → defeats rainbow tables and hides identical passwords.
- **Cost factor** = deliberate slowness (tune so a login takes ~100–250ms).
- Optional **pepper** = a server-side secret added before hashing; defends a DB-only leak.
- **Timing-safe compare** on verify.
- Menu: **bcrypt** (safe default) / **scrypt** / **argon2id** (best).
- Fast hashes (SHA-256, MD5) are **wrong** — an attacker hashes billions/sec.

**argon2 deep-dive.** **Memory-hard**: fills a large RAM block with data-dependent random reads and
can't free it mid-compute. This defeats **GPUs** — thousands of tiny cores, each with little memory;
64 MB × thousands of parallel guesses = far more RAM than the GPU has, so cores starve. Knobs: memory,
iterations, parallelism. Use **argon2id**.

> **One line:** hash (not encrypt) with unique salt + high cost + timing-safe compare; use
> bcrypt/argon2id — argon2 is memory-hard to defeat GPUs.

---

## C3 — JWT structure & attacks

**Diagram**
```
 xxxxx . yyyyy . zzzzz
   |       |       +- signature (proves not tampered)
   |       +- payload (READABLE, no secrets) {sub, exp, orgId, role}
   +- header {alg:RS256}  <- pin this on verify

 HS256: one shared secret, every verifier can also forge
 RS256: central signs (private key)
        buyer/warehouse verify (public key), cannot forge
```
**In plain words.** A sealed postcard: anyone can read it, but the seal proves who sent it and
that it was not changed. RS256: only central owns the stamp, others can only check it.

**Core idea.** `header.payload.signature`, base64url — **readable, not encrypted**. Put **no secrets**
in the payload.

- Claims: `sub`, `exp`, `iat`, `nbf`, `kid` + custom.
- **Alg-confusion attacks**: `alg:none`, or swapping RS256→HS256 so the public key is used as an HMAC
  secret. **Fix: pin the algorithm** on verify (`algorithms:['RS256']`).
- **HS256** (shared secret — verifier can also forge) vs **RS256** (private key signs, public key
  verifies — verifier **cannot** forge). RS256 is correct for **central signs / buyer+warehouse
  verify**.

> **One line:** JWT is readable (no secrets inside), signature proves integrity; pin the algorithm to
> stop alg-confusion; RS256 lets central sign and others verify without being able to forge.

---

## C4 — Access + refresh + rotation + theft detection

**Diagram**
```
 login -> access A1 (15m) + refresh R1
 R1 -> central -> A2 + R2   (R1 now SPENT)
 R2 -> central -> A3 + R3   (R2 now SPENT)

 attacker replays R1 (spent)
   v
 central: spent token reused = copied
   v
 revoke whole family (R3 dies too) -> both users re-login
```
**In plain words.** Access token = a day pass. Refresh token = a coupon to get the next day
pass, valid once. If someone uses a coupon that was already used, it was photocopied, so cancel
every coupon from that login.

- **Access token** — short (~15 min), stateless, sent on every request.
- **Refresh token** — long (~30 days), **stored at central** (so it's revocable), used **only** to get
  new access tokens.
- **Rotation** — each refresh is **single-use**; every refresh issues a **new** refresh token and
  invalidates the old one (a "family").
- **Reuse detection** — a **spent** refresh token reappearing is impossible in honest use → it means it
  was **copied** → **revoke the whole family** (both parties logged out; attacker locked out).
- **Client storage menu**: localStorage (XSS-readable ✗) / httpOnly cookie (XSS-safe, needs CSRF care) /
  **access in memory + refresh in httpOnly cookie** (common best).

> **One line:** short stateless access + long stored refresh; rotate refresh (single-use) so reuse of a
> spent token = theft → revoke the family; storage trades XSS vs CSRF risk.

---

## C5 — Session strategies
- **Server session** — id in cookie, truth on server → instant revoke, but a store hit per request and
  hard to share across services.
- **JWT in Authorization header** — stateless, scales across services, **no CSRF** — but revocation is
  hard (→ C4). **Your setup.**
- **Hybrid cookie** — JWT in an httpOnly cookie → stateless + XSS-safe, but owes CSRF defense.
- Microservices push you toward **stateless** (a shared session store is a bottleneck).
- "Session" ≠ "must be server-stateful" — a JWT is a session mechanism with state in the token.

> **One line:** server session (revocable, store-per-request), JWT header (stateless, scales,
> CSRF-free, hard to revoke), hybrid cookie (stateless + XSS-safe, needs CSRF) — pick per shape.

---

## C6 — Cookie flags
- **HttpOnly** — JS can't read it → stops **XSS theft**. Always on for auth cookies.
- **Secure** — HTTPS only → stops sniffing.
- **SameSite** — cross-site send rule → main **CSRF** defense. `Strict` / `Lax` (default) / `None`
  (requires Secure).
- **Signed** — tamper detection (a JWT is already signed).
- **Path / Domain** — scope; never set Domain to a parent you don't fully control.
- Recipe: `HttpOnly; Secure; SameSite=Lax(or Strict); ` + narrowest scope.

> **One line:** HttpOnly stops XSS theft, Secure stops sniffing, SameSite stops CSRF, signing detects
> tampering, tight scope limits exposure.

---

## XSS (taught before C7)

**Diagram: XSS vs CSRF**
```
 XSS : attacker script RUNS on your site
       evil input -> stored/reflected -> victim browser runs JS as victim
       stops: output-encoding, CSP, HttpOnly
 CSRF: attacker site makes the browser SEND a request with your cookie
       evil.com form -> POST yourbank.com (cookie auto-attached)
       stops: SameSite, CSRF token. Header-token auth is immune.
```
**In plain words.** XSS: the thief gets inside your house and acts as you. CSRF: the thief stays
outside and tricks you into posting the letter with your own stamp.

**Core idea.** Attacker-controlled input is rendered as **live JavaScript** in a victim's browser, on
your domain — so it runs **as the victim**.

- Source: unescaped stored/reflected input.
- Powers: read `localStorage` and non-httpOnly cookies (steal tokens), act as the user.
- Fixes: **output-encoding (#1)**, **CSP**, **httpOnly** cookies (contains damage).
- **XSS = code on your page** (HttpOnly + encoding + CSP stop it). Contrast CSRF below.

> **One line:** XSS runs attacker JS in the victim's browser as the victim — stop it with
> output-encoding, CSP, and httpOnly (which blocks token theft even if a script runs).

---

## C7 — CSRF
**Core idea.** The attacker tricks the **browser** into sending an authenticated request from another
site, riding the **auto-sent cookie** — no code on your site, nothing read.

- **Only applies to cookie auth.** Header-token auth is **immune** (the browser doesn't auto-add
  `Authorization`) → your JWT-in-header services are CSRF-free; only the hybrid-cookie path needs
  defense.
- Defenses: **SameSite** (simplest), **synchronizer token** (strongest), **double-submit cookie**
  (stateless).
- **HttpOnly does nothing for CSRF** (it never reads the cookie); **SameSite does nothing for XSS**
  (the script is already on your site) — opposite attacks, opposite defenses.

> **One line:** CSRF abuses auto-sent cookies to forge cross-site requests — cookie auth only
> (header tokens immune); defend with SameSite / synchronizer token / double-submit.

---

## C8 — CORS

**Diagram**
```
 browser page (a.com) --> fetch b.com/api
   | preflight OPTIONS
   v
 b.com answers Access-Control-Allow-Origin: a.com ?
   yes -> page may READ the response
   no  -> browser blocks the read
 curl / Postman / servers: no CORS check at all (auth protects the API)
```
**In plain words.** CORS is the browser protecting its user, not your server. Anyone outside a
browser ignores it, so real protection is auth.

**Core idea.** A **browser-only** rule: a page may not **read** another origin's response unless the
server opts in (preflight `OPTIONS` + allow-list).

- `credentials: true` (cookies) **forbids** `Origin: *` — must name the exact origin.
- **#1 misconception:** CORS protects **users in browsers**, **not the API**. curl/Postman/servers
  ignore it entirely. **Auth protects the API.**

> **One line:** CORS is a browser rule on reading cross-origin responses (protects users, not the API);
> curl ignores it; credentials forbid the * wildcard.

---

## C9 — RBAC / ABAC / scopes + ownership
- **RBAC** — by role (simple, ~90% of apps, coarse).
- **ABAC** — by attributes (region, time, amount; flexible, complex to audit).
- **Scopes** — per-token least privilege (great for API keys / OAuth / services). Often combined with
  RBAC.
- **All three miss OWNERSHIP.** They check the *type* of user, not whether the resource is *theirs*.
  Forgetting the ownership check = **IDOR** (change the id in the URL to reach others' data).
- Every resource access: **role/scope check AND ownership check** (`order.buyerId === req.user.id`).

> **One line:** model authz by role/attributes/scopes, but always ALSO check ownership — skipping it is
> IDOR.

---

## C10 — Multi-tenancy
- **`org_id` must come from the verified token** (`req.user.orgId`), never the request.
- **Every** query filters by `org_id`; one forgotten filter = **whole-tenant breach**.
- Don't rely on memory — enforce in a **shared repo layer** or **Postgres Row-Level Security (RLS)**.
- Models: **shared table + org_id** (default) / schema-per-tenant / DB-per-tenant.
- Test with **multiple tenants' data present** so a missing filter fails in dev.

> **One line:** isolate tenants by an org_id from the token (never the request), filter every query,
> and enforce it in a low layer (shared repo / RLS) — default to shared-table + org_id.

---

## C11 — OAuth2 / OIDC + PKCE

**Diagram**
```
 App                     Google (auth server)
  | redirect + client_id + code_challenge
  |------------------------------>|
  |          user logs in here    |
  |<-- redirect + short code -----|
  | server sends code + code_verifier
  |------------------------------>|
  |<-- access_token + id_token ---|
```
**In plain words.** You show your password only to Google. Google gives the app a one-time
ticket, and the app swaps the ticket plus a secret only it knows for real tokens.

- **4 roles:** resource owner (you), client (the app), authorization server (Google), + tokens.
- Password only entered at the **authorization server** — the client never sees it.
- Flow (**Authorization Code + PKCE**): app redirects to Google with `client_id`, `redirect_uri`,
  `scope`, and a **PKCE code_challenge** → you log in → Google redirects back with a **short-lived
  code** (loggable, useless alone) → app's **server** exchanges code + **code_verifier** (proving same
  app started & finished) for tokens.
- **OAuth2 = access** (`access_token`); **OIDC = identity** (`id_token`, a JWT). "Login with X" = OIDC.
- Use **Auth Code + PKCE** for all clients; **never** implicit or password flows (deprecated/unsafe).

> **One line:** OAuth2 delegates access, OIDC adds identity; Auth Code + PKCE logs you in at the
> provider, returns a useless short code, then swaps it (with a PKCE secret) for tokens server-side.

---

## C12 — OTP & CAPTCHA
**OTP** — adds a **possession** factor (password = knowledge, which leaks).
- **Random OTP** — server invents a code, stores it with expiry, sends via SMS/email (common in India).
- **TOTP** — both sides compute `HMAC(shared_secret, 30s time window)`; nothing sent; offline; changes
  every 30s.
- Must be **single-use + short-expiry** (replay) and **rate-limited on send AND verify** (flood +
  brute-force; a 6-digit code is only 1M options).

**CAPTCHA** — "is this a **human**?" (a bot filter, **not** auth).
- Old = puzzles (AI broke them); modern = **invisible behavioral scoring**, challenge only if
  suspicious. Verify the token **server-side**. Layer with rate-limits + OTP.

Factor map: password = **know**, OTP = **have**, biometric = **are**, CAPTCHA = **human?**

> **One line:** OTP proves possession (random-sent or TOTP-computed), single-use + rate-limited both
> ways; CAPTCHA is a human/bot filter (invisible scoring), verified server-side — not authentication.

---

## C13 — Invite / accept

**Diagram**
```
 admin invites -> row {token, email, role, org, expires, used:false}
                      v email link ?token=...
 invitee clicks -> exists? unexpired? unused?
                      v yes
 create account FROM THE ROW (never req.body.role) -> mark token used
```
**In plain words.** The token is a claim ticket. The role is written in the shop's own ledger,
not on the ticket, so the visitor cannot upgrade themselves.

- Long **random single-use token**, emailed to the address the **admin named**.
- Points to a **stored row** holding **email / role / org the admin chose**.
- On accept: verify **exists / unexpired / unused** → create the account from the **stored** values
  (never `req.body.role`) → **consume** the token.
- Same pattern powers **reset / verification / magic-link**.

> **One line:** invite = a single-use token to a stored row of admin-chosen values; on accept, create
> from the stored row (never the request) and consume the token.

---

## C14 — Password reset (safely)
Same token pattern as C13, plus three edges:
1. **Enumeration-safety** — always reply "if that email exists, a link was sent" (uniform response
   **and timing**), only send if real.
2. **Single-use, short-expiry (~30m), hashed token** in storage.
3. **Invalidate ALL sessions + refresh tokens on reset** — else the attacker who triggered it stays
   logged in. Optionally email a "password changed" alarm.

> **One line:** hashed single-use short-lived token + enumeration-safe replies + revoke every session
> on reset (or the reset is theater).

---

## C15 — Brute-force defense
- Attacks: **brute force** (one account, many guesses) and **credential stuffing** (leaked pairs).
- **Per-IP** limit catches one attacker → many accounts; **per-account** limit catches many attackers
  (botnet) → one account. **Need both.**
- **Escalate**, don't hard-lock: clean → **CAPTCHA** → **exponential backoff** → brief/auto-unlock
  lock. (A hard lock lets an attacker DoS a victim.)
- Rate-limiting covers **online** only; the **slow hash (C2)** covers **offline** (stolen DB). Add MFA.

> **One line:** track failures per-IP and per-account (each covers the other's blind spot), escalate
> friction rather than hard-lock, and remember rate-limits are online-only — the slow hash guards
> offline.

---

## C16 — Rate limiting (algorithms)

**Diagram**
```
 Fixed window  : |--60s--|--60s--|  100 at :59 + 100 at :00 = 200 in 2s
 Sliding window: always count the last 60s from now (no edge)
 Token bucket  : bucket refills at rate r; each request spends 1
                 bursts allowed, average capped
 Counts live in shared Redis; over limit -> 429 + Retry-After
```
**In plain words.** A bouncer with a clicker. The only question is when he resets it: on the
clock (fixed), never (sliding), or he hands out tokens that refill slowly (bucket).

Bouncer with a clicker; the only question is **when he resets it**.
- **Fixed window** — reset on the clock; simple but **boundary burst** (100 at 10:00:59 + 100 at
  10:01:00 = 200 in 2s) because the counter forgets everyone at once on reset.
- **Sliding window** — always count the **last 60s from now**; no reset edge to exploit. Common default.
- **Token bucket** — spend a token per request, bucket refills at a rate; **allows bursts, caps the
  average**.
- **Per-route** (tight login/OTP, loose reads); key by **IP and user**; counts in **shared Redis** (an
  in-memory per-instance counter is bypassable by spreading load across instances); return **429**
  before expensive work, with `Retry-After`.

> **One line:** fixed (boundary burst) / sliding (smooth default) / token bucket (bursty, capped);
> per-route, shared-Redis counts, 429 before expensive work.

---

## C17 — Service-to-service auth
**Core idea.** **User identity ≠ service identity.** Zero-trust: the internal network is not safe.

- **Service tokens** — each service carries its own central-issued, scoped, RS256 token. **Your
  natural fit** (you already issue RS256 JWTs).
- **mTLS** — both sides present certs; identity baked into the TLS handshake. Strongest, transparent
  via a service mesh, but cert ops.
- **HMAC-signed requests** — secret + timestamp → identity + integrity + anti-replay. The **webhook**
  pattern (D16).
- Two identities travel together (user token + service credential); **scope service tokens to least
  privilege**.

> **One line:** authenticate the calling service (not just the user) under zero-trust — service tokens
> (your fit), mTLS (strongest), or HMAC-signed requests (webhooks) — scoped least-privilege.
