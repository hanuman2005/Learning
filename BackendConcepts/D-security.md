# Section D — Security (OWASP-mapped)

> **The spine:** everything from outside is untrusted. D1–D7 are one disease — **data treated as
> code** — fixed by **separating data from code** and not trusting client-supplied privilege/identity.

## D1 — Input validation

**Diagram**
```
 BLACKLIST (bad)                 WHITELIST (good)
 block "<script>", "DROP"        allow ONLY: string, 1-50 chars, a-z0-9
 attacker finds a 3rd trick ->   anything else -> rejected
 gets in                         unknown attacks fail closed
 Always on the SERVER (curl skips your frontend JS).
 Check: type, range, format, allowed-set, SIZE.
```
**In plain words.** A club bouncer with a guest list, not a list of banned faces. If you are not
on the list you do not get in, even if nobody ever thought about you.

- **Whitelist, not blacklist** — define what's allowed, reject the rest; **fails closed** and defends
  attacks you haven't imagined. (Blacklist must predict every attack forever.)
- **Always server-side** — client checks are UX; an attacker uses curl and skips your JS.
- Validate **type, range, format, allowed-set, and SIZE** (a huge body/array is a DoS — A1).
- Validate **every source**: body, query, params, headers, cookies, uploads — every service.

> **One line:** whitelist + always server-side + type/range/format/size across every input source =
> the trust boundary that pre-empts most of D.

---

## D2 — Output encoding

**Diagram**
```
 user input  --D1 validate-->  stored  --D2 encode for context--> output
 value: <b>hi</b>
   in HTML body : &lt;b&gt;hi&lt;/b&gt;     (shown as text)
   in URL       : %3Cb%3Ehi%3C%2Fb%3E
   in JS string : \u003cb\u003ehi...
 Different place = different encoding.
```
**In plain words.** D1 checks what comes in. D2 makes sure that when the text is shown, the
browser treats it as text, not as code. A harmless value can be dangerous in the wrong place.

- **D1 guards IN; D2 guards OUT.** You need both — a valid value (`O'Brien`, `<3`) can be dangerous
  when rendered.
- Same value is **safe or deadly by context**: HTML body / HTML attribute / JavaScript / URL each need
  a **different** encoding. Encode for the **exact context**.
- Modern frameworks auto-encode until you opt out (`dangerouslySetInnerHTML`, `v-html`, string concat).
  Sanitize rich HTML with **DOMPurify**.
- **Backend job:** correct `Content-Type` + `X-Content-Type-Options: nosniff` so JSON isn't sniffed
  into executable HTML.

> **One line:** encode data for the exact context it lands in (HTML/attr/JS/URL differ); frameworks
> auto-encode until you opt out; backend sends honest content-type + nosniff.

---

## D3 — SQL injection

**Diagram**
```
 BAD : "SELECT * FROM users WHERE name='" + input + "'"
       input = ' OR '1'='1' --   -> query becomes: everyone
 GOOD: query  : SELECT * FROM users WHERE name = $1   (shape locked)
       values : [ input ]                             (only fills the hole)
```
**In plain words.** Gluing input into SQL lets the attacker rewrite your sentence. Sending SQL
and values separately means the value is only ever a name, never an instruction.

- Input concatenated into a query becomes **code** (`' OR '1'='1' --`).
- **Fix: parameterize** — SQL + values on **separate channels** (`$1` + values array); the parser locks
  the shape before the value arrives, so input can only fill a hole (B9).
- **ORMs parameterize by default**; **raw escape hatches** (`sql\`\``, `$queryRaw`, `.raw()`)
  reintroduce risk — bind there too.
- **Identifiers still need a whitelist** (B9). Defense in depth: least-privilege DB user, validation,
  hide DB errors from clients.

> **One line:** parameterize (separate SQL from values); ORMs do it by default but raw queries don't;
> whitelist identifiers; least-privilege DB user + hidden errors as backup.

---

## D4 — NoSQL injection (Mongo)

**Diagram**
```
 expected : { email: "a@b.com", password: "secret" }
 attack   : { email: "a@b.com", password: { "$ne": null } }
            -> Mongo reads "password NOT equal null" -> matches -> login!
 fix      : Joi.string() rejects the object  (or String(x))
```
**In plain words.** Mongo queries are objects, so the attacker sends a rule instead of a value.
Force the input to be a plain string and the trick is dead.

- Mongo queries are **objects**; send an **operator** where a string was expected (`{$ne:null}`,
  `{$gt:""}`) → matches anything. Also `$regex` DoS, `$where` runs JS (RCE).
- **Fix: type enforcement** — Joi `string()` rejects an object, or `String(x)` cast. Mongoose schemas,
  `express-mongo-sanitize` (strips `$`/`.` keys), never `$where` with input.

> **One line:** NoSQL injection sends a query-operator object where a string was expected — enforce the
> primitive type, use schemas, strip $ keys.

---

## D5 — Command injection

**Diagram**
```
 BAD : exec("convert " + filename)      filename = "a.png; rm -rf /"
                |
                v  shell reads ; and runs both commands
 GOOD: execFile("convert", [filename])  no shell -> ";" is just a character
 BEST: use a library, no shell at all
```
**In plain words.** A shell is a program that obeys symbols like `;` and `|`. Skip the shell and
those symbols lose their power.

- Input reaching a **shell** (`exec`) → `;` `|` `&&` chain OS commands (RCE).
- **Fix: skip the shell** — `execFile`/`spawn` with an **args array** (no shell parses metachars);
  never `shell:true`. **Better: use a library, not a CLI** — most apps never need `child_process`.
  Plus validate + least-privilege.

> **One line:** don't put input in a shell — use execFile/spawn with an args array, or better, don't
> shell out at all.

---

## D6 — SSRF (Server-Side Request Forgery)

**Diagram**
```
 attacker --> POST /fetch?url=http://169.254.169.254/ (cloud keys)
                 |
 YOUR SERVER fetches it from INSIDE the network -> returns secrets
 Defense: allow-list hosts, block private IPs on the RESOLVED address,
          http/https only, re-check redirects
 Fixed URL (KYC provider) = safe.  User-supplied URL = risk.
```
**In plain words.** You ask your server to make a phone call for you, and the attacker picks the
number: your own internal office line. The server is inside the walls, so it can reach things the
attacker cannot.

- Server fetches a **user-controlled URL** → attacker points it inward: cloud metadata
  `169.254.169.254` (IAM creds), internal admin, `localhost`, the DB.
- **Fix:** outbound **allow-list**, **block private/link-local IPs on the resolved address**, restrict
  scheme to http/https, re-check redirects.
- Rule of thumb: **fixed URL = safe** (your a KYC provider); **user-supplied URL = SSRF risk**.

> **One line:** SSRF makes your server fetch attacker-chosen internal URLs (esp. cloud metadata) —
> allow-list hosts, block private IPs on the resolved address, restrict scheme, check redirects.

---

## D7 — Path traversal

**Diagram**
```
 base = /app/uploads
 input = ../../etc/passwd
 path.resolve(base, input) = /etc/passwd
 check: result.startsWith('/app/uploads/') ?  NO -> reject
 Better: user sends file id, you map id -> real path
```
**In plain words.** `../` means "go up one folder". Resolve the final path first, then check it is
still inside the allowed room.

- `../` in a filename **climbs out** of the intended folder to `/etc/passwd`, `.env`, keys.
- **Fix:** `path.resolve(base, input)` then verify the result **startsWith the base dir** (check the
  **destination**, not the input). Map user input to **generated IDs**, never real paths. Azure Blob
  sidesteps local-FS traversal.

> **One line:** resolve the final path and confirm it stays inside the base dir; map input to IDs you
> control rather than filesystem paths.

---

## D8 — Security headers (Helmet)

**Diagram**
```
 response headers (Helmet):
   Strict-Transport-Security   -> always HTTPS
   X-Frame-Options / CSP       -> nobody can frame your page (clickjacking)
   X-Content-Type-Options      -> no content sniffing
   Referrer-Policy             -> do not leak URLs
   Content-Security-Policy     -> only whitelisted scripts run (XSS backstop)
```
**In plain words.** Headers are instructions your server gives the browser: "only HTTPS, do not
let strangers frame me, only run my own scripts". The browser then enforces them.

- **HSTS** (force HTTPS) / **X-Frame-Options** + `frame-ancestors` (clickjacking) / **nosniff** (D2) /
  **Referrer-Policy** (don't leak URLs) / **CSP** (whitelist script origins — strongest XSS backstop;
  fiddly, start report-only).

> **One line:** Helmet sets headers that force HTTPS, block framing, stop sniffing, limit referrer, and
> — via CSP — refuse non-whitelisted scripts (XSS backstop).

---

## D9 — CORS allow-list (security view of C8)

**Diagram**
```
 BAD : Access-Control-Allow-Origin: *            any site may read
 GOOD: Access-Control-Allow-Origin: https://app.example.com   only yours
 credentials:true + * = illegal
```
**In plain words.** Replace "everyone is welcome" with a named guest list.

- Replace `*` with named origins. CORS protects **users in browsers**, not the API. `*` forbidden with
  credentials. Action: swap `origin:'*'` for real frontend origins per service.

> **One line:** name your allowed origins (not *); CORS guards users not the API; * is illegal with
> credentials.

---

## D10 — Rate limiting (security view of C16/C15)

**Diagram**
```
 request -> [rate limiter: per route, per IP+user, Redis]
              under limit -> handler
              over limit  -> 429 + Retry-After (before expensive work)
 stops: brute force, stuffing, scraping, SMS-cost abuse, DoS
```
**In plain words.** A turnstile: it lets people through at a sane speed so nobody can flood the
building.

- Blunts brute force, credential stuffing, scraping, OTP/SMS-cost abuse, DoS. Per-route, key by IP+user,
  shared Redis, 429. Pair with slow hash (C2) + CAPTCHA (C12).

> **One line:** rate-limit per-route with shared-Redis counts against brute force / scraping / DoS, and
> pair it with slow hashing and CAPTCHA.

---

## D11 — Secrets management

**Diagram**
```
 code ------X------> secrets (never hardcode, never commit)
 .env (in .gitignore) -> env vars -> secrets manager (Key Vault)
 leaked in git?  deleting the commit is NOT enough -> ROTATE the secret
 give each service only the secrets it needs
```
**In plain words.** A secret in git is like a password shouted in a crowd. You cannot un-shout it,
so you change the password.

- **Never hardcode / commit** — `.env` in `.gitignore`; **git history is forever** → if leaked,
  **ROTATE** (don't just delete the commit).
- Homes: **env vars → secrets manager (Azure Key Vault) → vault**.
- **Rotate** periodically + on leak (C3 `kid`); **scope minimally** per service (C17).
- Secret-scanning (gitleaks) catches accidental commits.

> **One line:** keep secrets out of code and git (rotate if leaked), load from env/secrets-manager,
> rotate regularly, scope per service.

---

## D12 / D13 — Encryption at rest & in transit

**Diagram**
```
 AT REST (stored)                   IN TRANSIT (moving)
 DB/disk/backup: AES-GCM            client <-TLS-> server <-TLS-> service
 key in KMS, not beside the data    HSTS stops downgrade
 bank fields encrypted individually mTLS between services
 stops: stolen disk/dump            stops: someone listening on the wire
 Need BOTH. Neither covers the other.
```
**In plain words.** Locked safe (at rest) and armoured van (in transit). A good van does not help
if the safe is open, and the other way round.

- **At rest** (D12): **AES-GCM** (authenticated), key in a **KMS** (not next to the data), rotated;
  whole-disk automatic + **field-level** for bank data (N3). Protects stolen disk/backup/DB dump.
- **In transit** (D13): **TLS everywhere** (client + service-to-service), **HSTS** vs downgrade,
  **mTLS** (C17), cert pinning. Protects interception.
- Both needed — neither covers the other; keys via D11.

> **One line:** AES-GCM at rest with KMS-held keys (field-level for bank data) + TLS everywhere in
> transit (mTLS service-to-service, HSTS) — two states, two threats, both required.

---

## D14 — PII handling

**Diagram**
```
 request body {name, pan, bank, password}
   |-- log it whole? NO (biggest leak)
   v
 redact/mask: {name:"A***", pan:"ABCDE****F", password:"[REDACTED]"}
 minimize -> encrypt -> retention + delete -> access/delete rights
```
**In plain words.** The most common leak is not hackers. It is your own logs holding people's PAN
and passwords. Mask before you write.

- PII: name, phone, email, PAN, Aadhaar, bank, GSTIN (KYC is full of it). Legal duties: **DPDP Act
  (India)** + **GDPR**.
- **#1 leak = logging whole request bodies** → **redact/mask** before logging; never log
  passwords/tokens/full numbers; keep PII out of errors and URLs.
- Principles: **minimize, encrypt, retention + delete, data-subject rights** (design "delete this
  user" as a runnable op).

> **One line:** minimize + encrypt + redact-from-logs + retention/deletion + support access/delete
> rights; the biggest accidental leak is logging full request bodies.

---

## D15 — Audit logging

**Diagram**
```
 admin approves supplier
   v
 audit row: who=admin7 | what=APPROVE | target=sup42 | when | IP | req-id
            before={KYC pending}  after={verified}
 append-only, hash-chained, separate store, admin cannot edit it
 NOT the same as debug logs
```
**In plain words.** A CCTV recording that the cameraman cannot erase. It answers "who did what,
to what, when".

- Records **who / what / when / target / before / after** for sensitive actions (approvals, role
  changes, money, KYC, admin — N7).
- **≠ debug logs**: append-only, **tamper-evident** (hash-chain), **separate store/permissions** (an
  admin can't rewrite their own audit), **no secrets/PII inside**.
- Fields: actor, action, target, timestamp, IP, request id (→ H correlation IDs).

> **One line:** an immutable, tamper-evident, append-only record of who-did-what-to-what for sensitive
> actions, separate from debug logs and free of secrets.

---

## D16 — Webhook verification

**Diagram**
```
 provider --POST /webhook--> you
   raw body + header signature + timestamp
 1 compute HMAC(secret, RAW body) -> compare timing-safe
 2 timestamp within 5 min?  (stops replay)
 3 event id already processed? skip (idempotent)
 4 ack 200 fast, work async
```
**In plain words.** Anyone can post to your public URL. The signature is a wax seal only the real
sender can make. Check the seal on the exact bytes you received.

- A webhook is a **public** inbound POST — anyone can forge it.
- **Verify HMAC over the RAW body** (not `JSON.stringify(req.body)` — re-serializing breaks the
  signature), timing-safe. **Timestamp window** vs replay (C12). **Idempotent** processing keyed on
  the event id vs duplicate retries (F2).
- This is C17's signed-request pattern, **inbound**.

> **One line:** verify the HMAC over the raw body (timing-safe), enforce a timestamp window against
> replay, and process idempotently against duplicate retries.

---

## D17 / D18 / D19 — Dependency & supply-chain

**Diagram**
```
 your app = 5% your code + 95% dependencies (run with YOUR privileges)
 D17 known bugs   : npm audit / Snyk / Dependabot, fail CI
 D18 tampering    : lockfile + npm ci, watch typosquats (expresss)
 D19 container    : slim base image + Trivy scan, rebuild for patches
```
**In plain words.** You trust hundreds of strangers' code. Scan it for known holes, freeze exact
versions so nobody swaps it silently, and clean the box you ship it in.

- **Reality:** ~95% of your code is dependencies, running with your privileges.
- **D17 (known CVEs):** scan (`npm audit`, Snyk) + patch; automate with **Dependabot**; fail the build
  in CI.
- **D18 (supply-chain attacks):** typosquatting (`expresss`), hijacked maintainer, poisoned version →
  **commit a lockfile** for reproducible installs (integrity anchor), `npm ci` in CI, verify names.
- **D19 (containers):** base images carry CVEs → scan with **Trivy**, use **slim** images, rebuild for
  patches.

> **One line:** scan/patch known CVEs (audit/Snyk/Dependabot in CI), pin a lockfile against
> supply-chain tampering, and scan your container base image too.
