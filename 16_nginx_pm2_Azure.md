# nginx, pm2 and Azure: Deploying Node and Python Backends on a Linux VM

A working guide to putting a backend service on a Linux virtual machine in Azure, behind nginx, supervised by pm2 or systemd. It teaches the mental model first so you can diagnose a failing deployment, not only repeat a recipe.

**Audience:** backend engineers who can write a service and now need to ship, run and debug it in production.
**Prerequisites:** `03_Linux.md` (systemd, signals, networking, TLS, SSH, logs). Basic Node.js (`12_Node.js.md`) or Python (`04_Python.md`), and basic PostgreSQL (`09_PostgreSQL_and_Drizzle.md`).
**Scope:** a single-VM or small multi-VM deployment with nginx as reverse proxy, pm2 or systemd as supervisor, and Azure as the platform. Kubernetes and App Service are out of scope. AWS-focused material lives in the `CloudDevOps` folder, and this guide complements it: the concepts (VPC and VNet, security group and NSG, IAM and RBAC, instance role and managed identity) map closely, but names and defaults differ.
**Note on accuracy:** nginx and pm2 behaviour here is long-stable, but defaults and options vary by version, so confirm with `nginx -V`, the nginx documentation and `pm2 --version`. Azure service names, SKUs, tiers, limits, prices, portal wording and CLI syntax change often. Where this guide touches those, it describes the concept and tells you to confirm in current Microsoft Learn documentation. No prices or SKU lists are given on purpose. Hostnames such as `api.example.com` are placeholders.

## How to use this guide

Each concept chapter has four parts:

1. **Understand**: the mental model and the reasoning.
2. **Tools to master**: what to learn and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions to answer without notes.

Chapter 21 holds a deployment checklist, a rollback template and a 502 incident playbook. Chapter 22 lists labs, chapter 23 interview questions with model answers.

**What "good enough" means for a backend role:** you can provision a VM, put an app behind nginx with TLS, keep it alive and updatable without downtime, connect it to a managed database over a private path, see what it is doing, and recover from a bad deploy in minutes. Chapters 3, 4, 10, 12, 20 and 21 matter most.

## Contents

1. The deployment mental model
2. Part A: nginx architecture and configuration structure
3. Location matching and reverse proxying
4. Forwarded headers and trusting the proxy
5. Upstreams, load balancing and keepalive
6. Timeouts, buffering, streaming, WebSockets and body size
7. Compression, static files, caching and rate limiting
8. TLS, HSTS and security headers
9. Logs, operation and multiple apps on one host
10. Diagnosing nginx errors; nginx versus HAProxy, Caddy and cloud load balancers
11. Part B: pm2 fundamentals, ecosystem file and environment
12. Zero-downtime reload, logs, restarts and boot
13. pm2 versus systemd versus Docker, Python apps, multi-service layouts
14. Part C: Azure hierarchy, regions, zones and the Azure CLI
15. Linux VMs on Azure
16. Azure networking
17. Identity and access
18. Data services
19. Monitoring, cost control and security baseline
20. Reference deployment walkthrough, end to end
21. Deployment checklist, rollback template, 502 incident playbook
22. Practice labs
23. Interview questions
24. References

---

## 1. The deployment mental model

### Understand

A request crosses several independent layers, and each can fail on its own:

```
client -> DNS -> [Azure LB / NSG] -> VM NIC -> nginx (TLS, routing) -> app (pm2/systemd) -> database / cache
```

**Why a reverse proxy.** Your app server (Node, gunicorn) runs application code well. It is a poor place to terminate TLS, absorb slow clients, serve static files, enforce body limits or route several apps on port 443. nginx does these with a small event-driven core and lets the app listen on a loopback port that is never exposed.

**Why a process manager.** A bare `node server.js` dies with the terminal, a crash or a reboot. A supervisor starts it at boot, restarts it, captures output and applies limits. pm2 adds Node-specific features (cluster mode, graceful reload). systemd is the OS-level alternative (`03_Linux.md`, chapter 5).

**Why a platform layer.** The VM gives compute and a disk. Azure gives the network boundary (NSG), identity (managed identity instead of secrets), managed data services and monitoring. Treat the VM as replaceable and keep state in managed services.

**Debug outside-in.** Ask which hop first rejects the request: DNS, NSG, nginx, the app or the database. Give each concern one owner (for example, only nginx redirects to HTTPS), or debugging gets confusing.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Hop-by-hop testing | `curl -v`, `curl --resolve`, `curl http://127.0.0.1:PORT/health` on the VM | Test each hop separately |
| Sockets and supervisors | `ss -tulpn`, `systemctl status`, `pm2 ls` | Know who listens where and who owns each process |

### Common mistakes

- Debugging the app when the request never reached it.
- Exposing the app port publicly "temporarily" and forgetting it; keeping state on one VM's disk.

### Self-check

- Name the layers a request crosses and one failure mode of each.
- Why should the app bind to `127.0.0.1` when nginx is on the same host?

---

# Part A: nginx

## 2. nginx architecture and configuration structure

### Understand

**Process model.** One **master** process (root; reads config, binds privileged ports, manages workers) and several unprivileged **worker** processes that handle all connections. Each worker is **event-driven**: one thread multiplexes thousands of connections using `epoll` and never blocks on one slow client, so idle and slow connections are cheap.

- `worker_processes auto;` starts one worker per core. `worker_connections` (in `events`) caps connections per worker. A proxied request uses two (client side and upstream side), each a file descriptor, so raise `worker_rlimit_nofile` and the service's `LimitNOFILE` (`03_Linux.md`, chapter 4).
- On `nginx -s reload` the master parses the new config and starts new workers. Old workers finish current requests and exit, so connections are not dropped.

**Configuration structure.** Nested **contexts**; directives inherit downward unless overridden.

```nginx
user www-data;
worker_processes auto;
events { worker_connections 4096; }

http {
    include /etc/nginx/mime.types;
    include /etc/nginx/sites-enabled/*;      # Debian/Ubuntu; RHEL-family uses conf.d/

    server {                                  # virtual host
        listen 80;
        server_name api.example.com;
        location / { proxy_pass http://127.0.0.1:3000; }   # URL-space rule
    }
}
```

**main** holds process settings, **http** holds `upstream`, `map`, `log_format`, `limit_req_zone` and `server` blocks, a **server** is selected by `listen` and `server_name`, a **location** by URL, and **include** splices files so each app gets its own.

**Inheritance trap.** `add_header` and `proxy_set_header` are inherited only if the current level defines none of its own. One `add_header` in a `location` discards all `add_header` lines from `server` and `http`. Re-declare the full set or use an included snippet.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Validate and apply | `nginx -t`, `nginx -T`, `systemctl reload nginx` | Never reload without a passing test |
| Build and process facts | `nginx -V`, `ps -ef --forest \| grep nginx`, `journalctl -u nginx` | Check modules, workers, startup errors |

### Common mistakes

- Restarting without `nginx -t`, so a typo takes the site down.
- Duplicate `listen` and `server_name` pairs; low descriptor limits ("worker_connections are not enough").

### Self-check

- What do master and workers each do, and why does event-driven I/O matter?
- Why does one `add_header` in a `location` remove your security headers?

---

## 3. Location matching and reverse proxying

### Understand

**Server selection first.** nginx matches `listen` address and port, then the `Host` header against `server_name`. If nothing matches, the **default server** for that port answers. Define an explicit catch-all returning `444` so unknown hostnames never reach a real app.

**Location matching order.**

1. **Exact** `location = /path` wins immediately.
2. **Prefix** locations (plain or `^~`): nginx remembers the **longest** match. If it has `^~`, regex checks are skipped and it wins.
3. **Regex** locations (`~` case-sensitive, `~*` insensitive) are tested **in file order**; first match wins.
4. If no regex matched, the longest prefix is used.

```nginx
location = /health       { return 200 "ok\n"; }
location ^~ /static/     { root /var/www/app; }
location ~* \.(png|jpg)$ { expires 7d; }
location /api/           { proxy_pass http://api; }
location /               { proxy_pass http://web; }
```

A **regex can override a plain prefix**, which surprises people. Use `^~` to protect a prefix.

**`proxy_pass` and the trailing slash.** The most common cause of wrong upstream paths. With **no URI part**, the original request URI is forwarded unchanged. With a **URI part** (even `/`), the part of the request URI matching the location is replaced by it.

| Location | `proxy_pass` | Request | Upstream receives |
|---|---|---|---|
| `/api/` | `http://127.0.0.1:3000` | `/api/users` | `/api/users` |
| `/api/` | `http://127.0.0.1:3000/` | `/api/users` | `/users` |
| `/api` | `http://127.0.0.1:3000/` | `/api/users` | `//users` |
| `/api/` | `http://127.0.0.1:3000/v1/` | `/api/users` | `/v1/users` |

Keep the trailing slash consistent on both sides when you want prefix stripping. In regex locations or with variables no automatic replacement occurs (confirm in the `ngx_http_proxy_module` docs), so verify what the app receives. A literal hostname in `proxy_pass` is resolved once at startup or reload. If the IP can change, use a `resolver` and a variable. For a loopback app write `127.0.0.1`, not `localhost`, to avoid `::1` surprises.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Which location matched | `curl -v`, temporary `add_header X-Loc "api" always;` | Predict and confirm routing |
| What upstream received | app request logs, `tcpdump -i lo port 3000 -A` | Detect wrong path rewrites |
| Test before DNS | `curl --resolve api.example.com:443:<ip> https://api.example.com/` | Test a vhost pre-cutover |

### Common mistakes

- Missing or extra trailing slash giving `404` or `//path`.
- A regex location for static files capturing an API prefix; no default server.

### Self-check

- Which location handles `/static/logo.png` above? `/api/v1/x.png`?
- What does the upstream receive for `location /app/ { proxy_pass http://u/; }` and `/app/x`?

---

## 4. Forwarded headers and trusting the proxy

### Understand

Behind a proxy the app sees a connection from `127.0.0.1`, plain HTTP and the wrong `Host`. Without help it logs the wrong client IP, builds `http://` links and rate-limits everyone as one client. The proxy passes the facts as headers:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

`Host` is the hostname the client asked for (nginx's default sends the upstream name). `$proxy_add_x_forwarded_for` appends the client address to any existing chain (`client, proxy1, proxy2`). `X-Forwarded-Proto` drives redirects, secure cookies and generated URLs.

**Why the app must trust the proxy.** These are ordinary headers, so any client can forge them. The framework must be told **how many proxy hops to trust** so it reads the client from the right position in the chain and ignores forged entries. Trust everything and an attacker spoofs IPs to dodge limits. Trust nothing and every client looks like `127.0.0.1`.

- Express: `app.set('trust proxy', 1)`, then `req.ip`, `req.protocol` and `req.secure` use the headers. Fastify: the `trustProxy` option.
- Python: gunicorn `forwarded_allow_ips`; uvicorn `--proxy-headers` with `--forwarded-allow-ips`; Werkzeug `ProxyFix` with an explicit hop count. Verify names in each project's docs.

If a CDN or Application Gateway precedes nginx, raise the hop count or use nginx's `realip` module (`set_real_ip_from`, `real_ip_header X-Forwarded-For`) so `$remote_addr` is the true client. If nginx is the internet-facing edge, use `proxy_set_header X-Forwarded-For $remote_addr;` to discard client-supplied values.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Observe headers | debug endpoint echoing `req.headers`, `curl -H 'X-Forwarded-For: 1.2.3.4'` | Prove a forged header is ignored |
| Trust settings | Express `trust proxy`, Fastify `trustProxy`, `set_real_ip_from` | Configure the hop count |

### Common mistakes

- No `trust proxy`, so logs and limits see only `127.0.0.1`; trust set to `true` on an app also reachable without the proxy.
- Wrong `X-Forwarded-Proto`, causing redirect loops.
- App port open to the internet, so clients bypass nginx and forge every header.

### Self-check

- Why is `X-Forwarded-For` untrustworthy by default, and what does `trust proxy 1` mean?
- An app behind nginx redirects endlessly to HTTPS. What is likely wrong?

---

## 5. Upstreams, load balancing and keepalive

### Understand

```nginx
upstream api_backend {
    least_conn;
    server 10.0.1.4:3000 weight=2;
    server 10.0.1.5:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.6:3000 backup;
    keepalive 32;
}
server {
    location / {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

**Methods** (open-source nginx): round robin is the default and honours `weight`. `least_conn` suits requests of widely varying duration. `ip_hash` gives crude stickiness but breaks behind a shared NAT or CDN. `hash $key [consistent]` suits cache locality. `random [two]` suits many nginx instances sharing a pool. Prefer stateless apps (sessions in Redis or signed tokens) so stickiness is unnecessary.

**Health and retries.** `max_fails` failures within `fail_timeout` mark a server unavailable for that period. This is passive: nginx learns only from real traffic. Active probing is a commercial nginx Plus feature, so with open-source nginx use an external health system or a cloud load balancer. `proxy_next_upstream` decides which failures move a request to the next server (default: errors and timeouts). Non-idempotent requests such as `POST` are not retried once sent unless you opt in with `non_idempotent`, which risks duplicate side effects.

**Keepalive to upstreams.** By default nginx opens a new TCP connection per proxied request. `keepalive N` caches up to N idle connections per worker. It needs `proxy_http_version 1.1` and an empty `Connection` header, as shown. It lowers latency and `TIME_WAIT` and ephemeral-port pressure (`03_Linux.md`, chapter 7). The app's keep-alive timeout must be **longer** than nginx's idle use of a connection, or the app closes a connection nginx is about to reuse, causing sporadic 502s. Node's `server.keepAliveTimeout` defaults to 5 seconds, a known cause.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Which backend served | `$upstream_addr`, `$upstream_status` in logs | Attribute each request |
| Socket reuse and failover | `ss -tan state time-wait \| wc -l`, stop one backend and loop `curl` | Confirm keepalive helps; observe failover |

### Common mistakes

- `keepalive` without HTTP/1.1 and an empty `Connection` header (no effect).
- App keep-alive shorter than nginx's: "upstream prematurely closed connection".
- `ip_hash` behind a CDN; retries on non-idempotent endpoints.

### Self-check

- When choose `least_conn` over round robin?
- What do `max_fails` and `fail_timeout` do, and what do they lack versus active checks?

---

## 6. Timeouts, buffering, streaming, WebSockets and body size

### Understand

**Client-side timeouts:** `client_header_timeout`, `client_body_timeout` (gaps between reads), `send_timeout` (gap between writes to a slow client) and `keepalive_timeout` protect workers from slowloris-style holding. **Proxy-side timeouts**, each 60 seconds by default: `proxy_connect_timeout`, `proxy_send_timeout` and `proxy_read_timeout` (gaps between writes to, and reads from, the upstream). A handler that sends nothing within `proxy_read_timeout` yields **504**. It is a gap timer, not a total timer. Raising it hides slow endpoints, so prefer fixing the endpoint or moving long work to a queue and returning `202`.

**Buffering.** nginx reads the upstream response into buffers (`proxy_buffers`, `proxy_buffer_size`), spilling to a temp file if large, then sends at the client's pace, freeing the app from slow clients. Consequences:

- **Streaming** (SSE, chunked progress, LLM token streams) is held back until buffers fill. Use `proxy_buffering off;` in that location, or have the app send `X-Accel-Buffering: no`, with `proxy_http_version 1.1`, an empty `Connection` header and a long `proxy_read_timeout`.
- **Large response headers** beyond `proxy_buffer_size` give 502 with "upstream sent too big header". Increase `proxy_buffer_size`.

**WebSockets.** The handshake is HTTP with `Upgrade: websocket`. Hop-by-hop headers are not forwarded by default, so pass them and use HTTP/1.1:

```nginx
map $http_upgrade $connection_upgrade { default upgrade; '' close; }   # in http

location /ws/ {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_read_timeout 1h;
}
```

Idle WebSockets close after `proxy_read_timeout`, so use heartbeats shorter than it. With several upstreams, long-lived connections need a shared pub/sub backplane.

**Request body size.** `client_max_body_size` defaults to 1 MB; larger requests get **413** from nginx. The framework has its own limit and the smaller wins. For big uploads prefer direct-to-storage with a SAS URL (chapter 18).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Timing breakdown | `curl -w '%{time_connect} %{time_starttransfer} %{time_total}\n' -o /dev/null -s URL` | Locate where time goes |
| Streaming and limits | `curl -N URL`, `curl -T bigfile -X POST URL` | Verify SSE arrives incrementally; reproduce a 413 |

### Common mistakes

- Raising every timeout to hours.
- SSE that works locally but arrives all at once in production (buffering on).
- WebSocket 400 because Upgrade headers were not forwarded.

### Self-check

- Which timeout produces a 504, and is it a gap or total timer?
- Why does SSE need `proxy_buffering off`?

---

## 7. Compression, static files, caching and rate limiting

### Understand

**gzip.** `gzip on;` with `gzip_types application/json text/css application/javascript text/plain;`, a moderate `gzip_comp_level` (about 5) and `gzip_vary on;`. Text compresses well. Skip images, video and archives. It costs CPU.

**Static files.** nginx serves files far more efficiently than Node. `root` appends the location path to a directory. `alias` replaces the location prefix, a frequent source of 404s when slashes disagree. Give fingerprinted assets `expires 30d;` with `Cache-Control: public, immutable`.

**Caching has two meanings.** (1) Browser and CDN caching via `Cache-Control` and `ETag`: long lifetimes on content-hashed files, `no-store` or short on authenticated API responses. (2) nginx **proxy cache** (`proxy_cache_path` in `http`, `proxy_cache` in a location) for expensive, public, idempotent responses. The key defaults to scheme, host and URI, so include anything that changes the response. Caching authenticated or cookie-dependent responses in a shared cache leaks one user's data to another, a serious bug.

**Rate limiting.** nginx limits per key (usually client IP) with a **leaky bucket**:

```nginx
limit_req_zone  $binary_remote_addr zone=api:10m rate=10r/s;     # in http
limit_req_status 429;

location /api/ {
    limit_req  zone=api burst=20 nodelay;
    proxy_pass http://127.0.0.1:3000;
}
```

`rate=10r/s` is the sustained rate. `burst=20` allows excess requests beyond it: without `nodelay` they are **delayed** to match the rate, with `nodelay` they are served at once but use burst slots that refill at the rate. Beyond burst, requests are rejected (default 503, hence `limit_req_status 429`). `limit_conn` caps **concurrent** connections per key. Limits of the approach:

- **Behind a CDN or load balancer** all requests share a few proxy IPs, so everyone is throttled together. Recover the real IP with `realip` and trusted ranges (chapter 4), or limit at the edge (CDN, WAF, Front Door).
- Per-IP is not per-user. Per-user or per-key quotas belong in the app or an API gateway, with Redis as shared state. nginx state is **per instance**, and it protects against overload, not volumetric DDoS.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Compression and cache headers | `curl -H 'Accept-Encoding: gzip' -I URL`, `add_header X-Cache $upstream_cache_status;` | Prove what compresses and what is cached |
| Limit testing | `hey`, `wrk`, a `curl -s -o /dev/null -w '%{http_code}\n'` loop, `error.log` "limiting requests" | See 429 start where expected |

### Common mistakes

- `alias` and `root` confusion; long lifetimes on non-fingerprinted files.
- Caching authenticated responses at a shared layer.
- Limiting by `$remote_addr` behind a CDN; limiting the health-check path; expecting limits to be shared across VMs.

### Self-check

- What does `burst` do and how does `nodelay` change it?
- Why does per-IP limiting fail behind a CDN, and what are two fixes?
- Which responses must never enter a shared cache?

---

## 8. TLS, HSTS and security headers

### Understand

TLS fundamentals are in `03_Linux.md`, chapter 7. **Termination:** nginx holds the certificate and key, decrypts, and talks plain HTTP to the app on loopback. For a remote upstream use `proxy_pass https://...` and configure verification (`proxy_ssl_verify`, off by default).

```nginx
server { listen 80; server_name api.example.com; return 301 https://$host$request_uri; }
server {
    listen 443 ssl http2;                    # newer versions also offer "http2 on;"
    server_name api.example.com;
    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    add_header Strict-Transport-Security "max-age=31536000" always;
    server_tokens off;
}
```

**Protocols and ciphers, conceptually.** Allow TLS 1.2 and 1.3 only. TLS 1.3 has a small fixed set of strong suites. For 1.2 use vetted defaults (the Mozilla SSL Configuration Generator) rather than hand-written cipher strings, preferring forward secrecy and AEAD ciphers.

**Let's Encrypt with certbot.** An ACME client proves domain control and obtains a 90-day certificate. The common **HTTP-01** challenge needs the domain to resolve to this host and port 80 reachable, so the NSG must allow inbound 80 and 443. Follow current Certbot instructions for installation, then:

```
sudo certbot --nginx -d api.example.com
sudo certbot renew --dry-run
```

The `--nginx` plugin edits your server block; `certonly` only obtains the certificate. nginx must **reload** to use renewed files, so add `--deploy-hook "systemctl reload nginx"` if needed. Renewal runs from a systemd timer or cron job (`systemctl list-timers`). Monitor expiry independently, since a silent renewal failure becomes an outage in 90 days. Wildcards need DNS-01.

**HSTS** tells browsers to use only HTTPS for `max-age` seconds. Start short, confirm every subdomain supports HTTPS before `includeSubDomains`, and treat `preload` as nearly irreversible. **Security headers** worth setting: `X-Content-Type-Options: nosniff`, `Referrer-Policy`, framing control and a designed `Content-Security-Policy`. Use `always` so errors carry them, and do not duplicate headers the app sets. `server_tokens off;` hides the version, a small leak reduction and no substitute for patching.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspect served cert | `openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null \| openssl x509 -noout -dates -subject -issuer` | Read expiry, subject, chain |
| Protocol, redirect, headers | `openssl s_client -tls1_2`, `curl -I http://...`, `curl -I https://...` | Prove old protocols are refused, one redirect, correct HSTS |
| Certificates | `certbot certificates`, `certbot renew --dry-run` | Verify renewal before it matters |

### Common mistakes

- Closing port 80 after issuing, which breaks HTTP-01 renewal; renewing files but never reloading nginx.
- Serving `cert.pem` instead of `fullchain.pem` (missing intermediate).
- Careless HSTS `preload` or `includeSubDomains`.

### Self-check

- Why must port 80 stay reachable for HTTP-01 renewals?
- What does HSTS protect against, and what is the risk of a long `max-age` set too early?

---

## 9. Logs, operation and multiple apps on one host

### Understand

**Two logs.** `access.log` (one line per request) and `error.log` (nginx's own problems and upstream failures), by default in `/var/log/nginx/`. The package's `logrotate` file signals nginx to reopen files. The default format lacks timing, so add it:

```nginx
log_format timed '$remote_addr [$time_local] "$request" $status $body_bytes_sent '
                 'rt=$request_time uct=$upstream_connect_time urt=$upstream_response_time '
                 'ua=$upstream_addr us=$upstream_status rid=$request_id';
access_log /var/log/nginx/api.access.log timed;
```

`$request_time` covers the whole request including slow client transfer. `$upstream_response_time` is the app's time. If `rt` far exceeds `urt` the client is slow, and if `urt` is large the app is slow. A large `$upstream_connect_time` points to a saturated listen queue or network trouble. `$upstream_addr` and `$upstream_status` list several values after retries. Forward `$request_id` (`proxy_set_header X-Request-ID $request_id;`) and log it in the app to join logs across layers.

**`nginx -t`, reload, restart.** `nginx -t` checks syntax and files, not behaviour. **Reload** is graceful, and an invalid config leaves the old one running. **Restart** drops connections and is needed only for binary upgrades or some module changes. Sequence: edit, `nginx -t`, reload, `curl`, watch `error.log`.

**Multiple apps.** Two mechanisms combine: **`server_name`** (hostname per app, preferred: independent TLS, headers and limits) and **path prefixes** (several apps under one hostname, needing the trailing-slash care of chapter 3).

```nginx
server { listen 443 ssl; server_name api.example.com;   location / { proxy_pass http://127.0.0.1:3000; } }
server { listen 443 ssl; server_name admin.example.com; location / { proxy_pass http://127.0.0.1:3100; } }
server { listen 443 ssl default_server; server_name _;  ssl_reject_handshake on; return 444; }
```

With SNI nginx picks the certificate by hostname, so each hostname needs a matching certificate. `server_name` precedence is exact, then longest wildcard, then regex. Use one file, one log and one distinct loopback port per app with a documented port map (clashes cause `EADDRINUSE`), and a DNS A record per hostname.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Config and logs | `nginx -t`, `nginx -T \| grep -n server_name`, `tail -F /var/log/nginx/*.log` | Test before reload; find conflicting names |
| Log analysis | `awk '{print $9}' access.log \| sort \| uniq -c`, `awk` on `rt` and `urt`, `goaccess` | Summarise status codes, find slow endpoints |
| Vhost testing | `curl --resolve host:443:IP https://host/`, `logrotate -d` | Test each vhost before DNS; confirm rotation |

### Common mistakes

- No timing in the log, so latency cannot be attributed; rotation without reopen, so nginx writes to a deleted file (`03_Linux.md`, chapter 2).
- Treating `nginx -t` success as proof the site works; tokens in query strings landing in logs.
- No default server, so an unknown hostname serves another app.

### Self-check

- What does `rt` 8 s with `urt` 0.05 s tell you?
- How does nginx choose a server block, and what if nothing matches?

---

## 10. Diagnosing nginx errors; nginx versus HAProxy, Caddy and cloud load balancers

### Understand

Read `error.log` first, then access-log status and timing, then test the upstream directly from the VM.

| Status | Meaning | Typical causes |
|---|---|---|
| **502** | no valid upstream response | app down, wrong port or bind address, connection closed early, header too large, SELinux |
| **503** | refused or none available | `limit_req` default status, all upstreams marked down |
| **504** | upstream too slow | handler beyond `proxy_read_timeout`, blocked event loop, saturated DB pool |
| **499** (access log only) | client closed before nginx replied | client or outer load balancer timeout, slow endpoint |
| **413** | body over `client_max_body_size` | uploads |
| **404 from nginx** | no matching file or location | wrong `root` or `alias`, trailing slash |

**Error-log messages.**

- `connect() failed (111: Connection refused) while connecting to upstream`: nothing listens there. Check `ss -tulpn`, `pm2 ls`, `127.0.0.1` versus `::1`, port. `(13: Permission denied)`: SELinux network-connect policy or socket permissions.
- `upstream prematurely closed connection while reading response header from upstream`: the app crashed, was OOM-killed, or closed an idle keepalive connection. Correlate with app logs and `dmesg`. `recv() failed (104: Connection reset by peer)` is similar.
- `upstream timed out (110: ...) while reading response header`: the 504 case.
- `upstream sent too big header`: raise `proxy_buffer_size` or shrink cookies. `no live upstreams`: every server in the group is marked failed.

**Wrong `proxy_pass` path.** Symptom: the app returns its own 404 for URLs that work directly. Compare the intended path with what nginx forwards (`tcpdump -i lo -A port 3000`), then fix the trailing slash.

**Procedure for a 5xx.** (1) Reproduce with `curl -v`; is nginx or the app answering? (2) Read `error.log` at that timestamp; the message names the phase. (3) Call the app directly on loopback: if it fails the app or a dependency is at fault, if it works look at nginx config or the upstream address. (4) Check `pm2 ls` or `systemctl status`, restart counts, `dmesg` for OOM, `urt` and `uct` in the log, and limits (`ss -s`, `ls /proc/<pid>/fd | wc -l`). (5) Fix the cause. Do not just raise timeouts.

**nginx versus alternatives.**

| Tool | Strength | Trade-offs | Choose when |
|---|---|---|---|
| **nginx** | web server plus reverse proxy, static files, caching | active health checks are commercial, verbose config | default for a VM with a few apps |
| **HAProxy** | strong L4 and L7 balancing, rich health checks and stats | not a static server or cache | dedicated balancing tier |
| **Caddy** | automatic HTTPS, concise config | smaller ecosystem | small deployments valuing simplicity |
| **Cloud load balancers** (chapter 16) | managed, zone-redundant, probes, WAF options | cost, another hop and timeout layer | several VMs, HA, WAF, global entry |

They combine: a cloud load balancer in front of several VMs, each running nginx for TLS, static files and per-app routing. Keep timeouts consistent: an outer layer's idle timeout must exceed the longest legitimate request, or you see 499 at nginx and 502 or 504 at the outer layer.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Error evidence | `grep -E 'upstream\|connect\(\)' error.log`, `dmesg -T \| grep -i -E 'killed process\|out of memory'` | Map each message to a cause; prove an OOM |
| Direct test | `curl -v http://127.0.0.1:3000/health` | Separate nginx from app faults |
| Sockets and packets | `ss -tulpn`, `tcpdump -i lo -nn -A port 3000` | See listeners, prove what nginx sent |

### Common mistakes

- Raising `proxy_read_timeout` for every 504; reading only the access log.
- Forgetting 499 is the client's choice, often a shorter outer timeout.

### Self-check

- A 502 appears only right after deploys. Name two likely causes.
- What does 499 mean and why is it not an nginx failure?
- The app returns 404 only through nginx. What do you check?

---

# Part B: pm2

## 11. pm2 fundamentals, ecosystem file and environment

### Understand

**What pm2 is.** A process manager for Node.js (it can run other interpreters). A background daemon supervises apps, restarts crashes, keeps a process list, captures logs, offers reloads and runs multiple instances. State lives in `~/.pm2` of the user who runs it, so **each Unix user has a separate daemon and list**.

**Fork mode (default).** One child process, one core, any language: `pm2 start app.py --interpreter python3`.

**Cluster mode** (`-i N`, `exec_mode: 'cluster'`) is Node-only. pm2 uses Node's `cluster` module: a master forks N workers **sharing one port**, with connections distributed round-robin on Linux. A Node process runs JavaScript on one thread, so N workers use N cores and survive one worker crashing. Implications:

- The app must be **stateless** across workers. In-memory sessions, caches, counters and rate-limit state are per worker. Use Redis or a database.
- **Scheduled jobs** (`node-cron`) run in every worker. Run them in one fork-mode process, a systemd timer, or behind a lock.
- Database connections multiply: workers times pool size must stay below PostgreSQL `max_connections`.
- `instances: 'max'` means one per core. Leave headroom for nginx and the OS. On a one- or two-core VM cluster mode gains little.

**Ecosystem file.** Declare apps in a version-controlled file rather than long CLI invocations:

```js
module.exports = {
  apps: [{
    name: 'api',
    script: 'dist/server.js',
    cwd: '/opt/app/current',
    instances: 2,
    exec_mode: 'cluster',
    max_memory_restart: '600M',
    kill_timeout: 10000,
    wait_ready: true,
    listen_timeout: 15000,
    min_uptime: '10s',
    max_restarts: 10,
    exp_backoff_restart_delay: 100,
    out_file: '/var/log/app/api.out.log',
    error_file: '/var/log/app/api.err.log',
    merge_logs: true,
    time: true,
    env: { NODE_ENV: 'development' },
    env_production: { NODE_ENV: 'production', PORT: 3000 },
  }],
};
```

Start with `pm2 start ecosystem.config.js --env production`. Confirm every option name and default (including `exp_backoff_restart_delay`, in milliseconds) in the pm2 documentation for your version. Core commands: `pm2 ls` (status, restarts, uptime, memory, CPU), `pm2 describe api`, `pm2 logs api`, `pm2 monit`, `pm2 restart api` (stop then start, brief gap), `pm2 reload api` (rolling in cluster mode, chapter 12), `pm2 stop/delete api`, `pm2 save`.

**Environment handling.** `env` holds defaults and `env_<name>` overrides with `--env <name>`. **pm2 caches the environment** at first start, so a plain `pm2 restart` does not apply changed values; use `--update-env` or delete and start again. Keep secrets out of the ecosystem file: use a `600` env file owned by the service user, deploy-time injection, or fetch from Key Vault at startup with the VM's managed identity (chapter 17). `cwd` anchors relative paths, and with a symlinked release layout you must reload after switching the symlink. For Python workers set `interpreter: '/opt/app/.venv/bin/python'` explicitly, since pm2 does not activate virtualenvs.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Process list and live view | `pm2 ls`, `pm2 describe`, `pm2 jlist`, `pm2 monit`, `pm2 logs` | Read status, restarts, uptime, memory during a deploy |
| Config as code | `ecosystem.config.js`, `--env`, `--update-env` | Reproduce a deployment from the repo |
| Env inspection | `pm2 env <id>`, `tr '\0' '\n' < /proc/<pid>/environ` | See what the process really received |

### Common mistakes

- Running pm2 as root once and as a user later: two process lists.
- Cluster mode with in-memory state; scheduled jobs running once per worker.
- Expecting `pm2 restart` to pick up new variables; committing secrets in the ecosystem file.

### Self-check

- What does cluster mode do at the Node level, and what state breaks?
- Why did a new variable not appear after `pm2 restart`?

---

## 12. Zero-downtime reload, logs, restarts and boot

### Understand

**`restart` versus `reload`.** `restart` stops then starts, leaving a gap. `reload` in **cluster mode** replaces workers one at a time: start a new worker, wait until ready, stop an old one. In fork mode `reload` behaves like restart.

**Requirements for genuine zero downtime.**

1. **Cluster mode, at least 2 instances.** With one instance nothing serves while it starts.
2. **Readiness signal.** By default pm2 considers a worker ready once it listens or after `listen_timeout`. If startup does work around that (pools, warm-up), set `wait_ready: true` and call `process.send('ready')` when the app can serve.
3. **Graceful shutdown.** pm2 sends **`SIGINT`** first, then `SIGKILL` after `kill_timeout` ms. Handle `SIGINT` and `SIGTERM` (systemd and Docker send `SIGTERM`): stop accepting, finish in-flight requests, close pools, exit.

```js
const server = app.listen(PORT, () => { if (process.send) process.send('ready'); });

let shuttingDown = false;
function shutdown() {
  if (shuttingDown) return;
  shuttingDown = true;
  server.close(async () => { await db.end(); process.exit(0); });
  setTimeout(() => process.exit(1), 8000).unref();   // before kill_timeout
}
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);
```

4. **`kill_timeout` above your drain time.** The default is short (documented as 1600 ms). Idle keep-alive sockets can stall `server.close`; close idle connections (`server.closeIdleConnections()` on recent Node).
5. **Backward-compatible changes.** Old and new code serve together during a reload. Use expand-then-contract migrations (`09_PostgreSQL_and_Drizzle.md`).
6. **Health-gated deploy script.** After reload, poll `/health` through nginx and roll back on failure. Long-lived connections (WebSockets, SSE) drop on worker stop and must reconnect.

**Deploy sequence on one VM.** Upload to `releases/<id>`, install and build, run expand migrations, switch the `current` symlink, `pm2 reload ecosystem.config.js --env production --update-env`, check health, keep the previous release for rollback.

**Logs.** By default `~/.pm2/logs/<name>-out.log` and `-error.log`, per instance in cluster mode unless `merge_logs: true`. **They grow without limit**, so install rotation (`pm2 install pm2-logrotate`, then `pm2 set pm2-logrotate:max_size 50M` and `:retain 14`; confirm option names in its README) or use `logrotate` with `copytruncate` (`03_Linux.md`, chapter 8). Prefer structured JSON logs with a request ID matching nginx `$request_id`.

**Auto-restart and crash loops.** `autorestart` is on by default. It survives transient faults but **masks crash loops**: a service dying seconds after start can restart hundreds of times, hammering the database and flooding logs while `pm2 ls` shows "online". Controls:

- `min_uptime`: stopping sooner counts as an unstable restart. `max_restarts`: after this many, pm2 marks the app `errored`. `restart_delay` is a fixed wait and `exp_backoff_restart_delay` an exponential backoff from the given milliseconds.
- `max_memory_restart` is a **safety net, not a fix**. A steadily growing process is a leak: bound the damage, track restarts and find the leak with heap snapshots (`03_Linux.md`, chapter 6).
- Alert on restart count and uptime, not only "online".

**Start on boot.** `pm2 startup systemd -u deploy --hp /home/deploy` prints a command to run with sudo, which installs a unit (`pm2-<user>.service`) that runs `pm2 resurrect`. `pm2 save` writes the current list to `~/.pm2/dump.pm2`. Forgetting `pm2 save` after changing apps is the classic "did not come back after reboot". Re-run `pm2 startup` after the Node path changes and `pm2 update` after upgrading pm2. Test by rebooting a non-production VM.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Reload proof | `curl` loop during `pm2 reload`, count non-200 | Show zero failed requests |
| Signals and timing | `process.on('SIGINT'/'SIGTERM')`, `wait_ready`, `listen_timeout`, `kill_timeout` | Drain cleanly; tune to startup and drain time |
| Logs and loops | `pm2-logrotate`, `min_uptime`, `max_restarts`, `pm2 jlist \| jq` | Prove logs cannot fill the disk; make crash loops visible |
| Boot | `pm2 startup`, `pm2 save`, `systemctl status pm2-<user>` | Survive a tested reboot |

### Common mistakes

- `reload` on a single instance or in fork mode; no signal handler; `kill_timeout` shorter than the drain.
- Sending `ready` before the DB pool connects; breaking schema changes shipped with the code.
- No log rotation; treating `max_memory_restart` as the fix for a leak; `pm2 startup` without `pm2 save`.

### Self-check

- Which signal does pm2 send first, and what happens after `kill_timeout`?
- Why does a one-instance `reload` still cause downtime?
- How does auto-restart mask a crash loop, and which settings expose it?

---

## 13. pm2 versus systemd versus Docker, Python apps, multi-service layouts

### Understand

pm2's unique value is cluster mode and rolling reload for Node. systemd is the more robust supervisor (cgroup limits such as `MemoryMax`, sandboxing, ordering, journald) and pm2 itself needs it at boot. Docker packages the runtime so the host does not define it, and normally runs one process per container with a restart policy or an orchestrator. Reasonable choices: pm2 cluster for a Node API wanting zero-downtime reloads on one VM, plain systemd (`Restart=on-failure`) when clustering is unnecessary, containers when environment reproducibility matters most. Do not stack supervisors that fight over a process (pm2 inside a container that also has a restart policy).

**Python: prefer systemd plus gunicorn.** Python web apps need an application server. **gunicorn** (WSGI, or with uvicorn workers for ASGI) manages worker processes, graceful worker replacement on `SIGHUP` and timeouts:

```ini
[Service]
User=app
WorkingDirectory=/opt/pyapp/current
EnvironmentFile=/etc/pyapp/env
ExecStart=/opt/pyapp/venv/bin/gunicorn app.wsgi:application --bind 127.0.0.1:8000 --workers 3 --graceful-timeout 30 --forwarded-allow-ips=127.0.0.1
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=3
TimeoutStopSec=40
```

`systemctl reload` sends `SIGHUP`; check gunicorn's docs for signal semantics and `--preload` trade-offs. pm2 with `interpreter` suits simple Python workers, but cluster mode does not apply. Many teams use systemd for everything and pm2 only where Node cluster reload is valued.

**Multi-service layout.** One VM, one nginx:

```
nginx :443 -> api.example.com   -> 127.0.0.1:3000 (pm2 'api', cluster x2)
           -> admin.example.com -> 127.0.0.1:3100 (pm2 'admin', fork)
           -> ml.example.com    -> 127.0.0.1:8000 (systemd gunicorn)
           (worker, scheduler: separate pm2 fork apps or systemd timers; no port)
```

Each app gets its own loopback port (none public), service user where practical, logs and a documented port map. Memory is shared, so sum instance limits and keep OS headroom, or one leak OOM-kills a neighbour. Run queue consumers and cron as separate processes, not extra duties of the web process.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Units and ownership | `systemctl cat`, `systemctl list-units --type=service`, `journalctl -u`, `pm2 ls` | Write a gunicorn unit from memory; say who owns each process |
| Python servers | `gunicorn --check-config`, `--workers`, `--timeout` | Size workers |

### Common mistakes

- Running a development server (`python app.py`) in production.
- Two supervisors fighting over a port; no per-service memory budget.
- Cron or queue workers started by every instance.

### Self-check

- Give a case for systemd over pm2 and one for pm2 over systemd.
- How do you gracefully reload a gunicorn service under systemd?

---

# Part C: Azure for backend engineers

> Confirm service names, tiers, limits and CLI syntax in current Microsoft Learn documentation before relying on them. Microsoft renames and retires services, and portal wording changes. Concepts below are stable, specifics are flagged.

## 14. Azure hierarchy, regions, zones and the Azure CLI

### Understand

**Hierarchy.** **Tenant** (the Microsoft Entra ID directory, the identity boundary) contains optional **management groups**, then **subscriptions** (billing, quota and access boundary; many teams separate production and non-production), then **resource groups** (containers for resources sharing a lifecycle: a VM, its disk, NIC, NSG, public IP; deleting the group deletes its contents), then **resources**, identified by `/subscriptions/<id>/resourceGroups/<rg>/providers/<provider>/<type>/<name>`. RBAC, policy, tags and budgets attach at any scope and are **inherited downward**. Use **tags** (`env`, `owner`, `app`) for cost attribution.

**Regions and availability zones.** Choose a **region** by user latency, data residency, service and VM-size availability and price. Many regions offer **availability zones**: physically separate datacentres with independent power and network. Spreading VMs across zones survives a single-datacentre failure. Keep app, database and cache in the **same region** to avoid latency and egress charges. Redundancy ladder: one VM (no protection beyond platform recovery; patching and host maintenance cause downtime) < several VMs across zones behind a load balancer < multi-region with global routing. Match the investment to the availability target.

**The Azure CLI** (`az`) is a client for the Azure Resource Manager API. Everything the portal does can be scripted, and scripts are reviewable and repeatable. Commands follow `az <group> <subgroup> <verb> --options`. Workflow:

1. **Authenticate and select scope:** `az login`, `az account show`, `az account set --subscription <id-or-name>`. Confirm the subscription before any destructive command. CI uses a service principal or workload identity, Azure-hosted automation a managed identity.
2. **Create a resource group** (`az group create --name <rg> --location <region>`) and then resources (`az network ...`, `az vm create ...`, `az postgres flexible-server create ...`). Use `--help` and the CLI reference for exact parameters.
3. **Inspect and operate** with `az vm show`, `az vm list -d`, `--query` (JMESPath), `-o table|json|tsv`, `az vm deallocate`, `az vm start`.
4. **Tear down** with `az group delete`, which removes everything in the group, so check the name.

```bash
az group create --name rg-app-prod --location <region>
az vm list -g rg-app-prod -d --query "[].{name:name, ip:publicIps, state:powerState}" -o table
az vm deallocate -g rg-app-prod -n vm-app-01
```

For scripts use `set -Eeuo pipefail` (`03_Linux.md`, chapter 9), `-o tsv` with `--query` to capture single values, idempotent steps, no secrets on command lines (shell history), `--no-wait` for long operations, and tags at creation. For larger estates move to infrastructure as code (Bicep or Terraform), which shows a plan before applying. Parameter names, image aliases and extension requirements change between versions, so treat snippets here as shapes to confirm.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Scope and context | Azure Portal, `az account show/set`, `az group list` | Never act on the wrong subscription |
| Querying and scripting | `az ... --query`, `-o tsv`, `jq`, `az <group> --help` | Extract any field; write an idempotent provisioning script |
| Region choice | Microsoft Learn region docs, `az vm list-sizes --location` | Verify a size exists in the region |

### Common mistakes

- Working in the wrong subscription; `az group delete` on the wrong group.
- Unrelated resources in one group; app and database in different regions.
- Assuming one VM equals high availability; secrets as command arguments.

### Self-check

- What do a tenant, subscription and resource group each control, and what does an availability zone protect against?
- Why prefer scripts or IaC to portal clicks for production?

---

## 15. Linux VMs on Azure

### Understand

**Size (conceptual).** A size fixes vCPUs, memory, temp disk, maximum disk IOPS and throughput, network bandwidth and NIC count. **Families** target general purpose, compute, memory, storage or burstable workloads. Burstable sizes accrue CPU credits when idle and throttle under sustained load, a classic sudden-slowness cause for a busy API. Size from measurements, then right-size.

**Disks.** The OS and data disks are **managed disks**, with performance caps per disk **and** per VM size. Hitting either looks like slow I/O with idle CPU (`03_Linux.md`, chapter 6). Many sizes have a **temp disk** (often `/mnt`) that is local and **not persistent**. Never keep data there.

**Images and SSH keys.** Start from a supported Ubuntu LTS image (or a custom or gallery image) and pin versions. Create VMs with **key authentication** and disable passwords (`03_Linux.md`, chapter 12): generate `ssh-keygen -t ed25519` locally and supply the **public** key at creation. The admin username you choose is your login.

**cloud-init.** Linux images run a cloud-init document passed as custom data on first boot, making a VM reproducible (`#cloud-config` with `packages:` and `runcmd:`). Its output is in `/var/log/cloud-init-output.log`. Custom data is not a secret store.

**Public versus private IPs.** Every NIC has a **private IP** from its subnet. A **public IP** is a separate, optional resource. Prefer a public IP only on the load balancer or one entry point, private IPs for app and data tiers, and administration through Azure Bastion, a jump host or VPN. Public IP SKUs have changed (Standard is the current direction, Basic has been retired; confirm). A dynamic public IP can change on deallocation and break DNS.

**Billing states.** Stopping from inside the OS leaves compute allocated and **billed**. **Deallocating** (`az vm deallocate`, portal "Stop") stops compute charges; disks and static IPs still bill.

**Other facts.** The Azure Linux agent (`waagent`) handles provisioning and extensions. Instance metadata is at `169.254.169.254` and is how managed identity tokens are obtained. Planned host maintenance can reboot a VM, so supervisors must start on boot. **Boot diagnostics** and the **serial console** recover a VM broken by a bad `sshd_config` or `fstab`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Provisioning | Portal, `az vm create`, cloud-init | Create a reproducible key-auth VM |
| Inspect | `az vm show -d`, `lsblk`, `df -h`, `cloud-init status` | Find IPs, size, state; tell OS, data and temp disks apart |
| Recovery | boot diagnostics, serial console | Fix a non-booting or unreachable VM |

### Common mistakes

- Data on the temp disk; secrets in cloud-init custom data.
- Password auth on, or SSH open to `0.0.0.0/0` on a public IP.
- Believing "stopped" means not billed; burstable size for sustained load.

### Self-check

- What is the difference between stopping from the OS and deallocating?
- Name two reasons a VM is slow with idle CPU.

---

## 16. Azure networking

### Understand

**VNet and subnets.** A VNet is your private address space (for example `10.0.0.0/16`) in a region, split into **subnets**. Resources reach each other by private IP by default. Plan non-overlapping ranges for later **peering** or on-premises links. Some services need a **delegated or dedicated subnet** (private-access PostgreSQL, Application Gateway).

**Network security groups (NSGs).** Stateful filters attached to a **subnet and/or NIC**. Each rule has a priority (100 to 4096, **lowest number evaluated first**), direction, source, destination, port, protocol and allow or deny. First match wins. Default rules (65000 and above) allow VNet-internal traffic and load balancer probes, **deny all other inbound**, and allow outbound to the internet.

| Priority | Name | Port | Source | Action |
|---|---|---|---|---|
| 100 | allow-ssh-admin | 22 | office or VPN range | Allow |
| 110 | allow-http | 80 | Internet | Allow |
| 120 | allow-https | 443 | Internet | Allow |
| 65500 | DenyAllInbound (default) | any | any | Deny |

Never leave SSH open to `Any` on a public IP; use Bastion, VPN, just-in-time access or a restricted range. With NSGs on both subnet and NIC, traffic must pass both. NSGs complement the host firewall and app binding: check all three (`03_Linux.md`, chapter 7). **Application security groups (ASGs)** label NICs so rules say "web to db" rather than listing IPs. NSG **flow logs** record accepted and denied flows.

**Outbound and NAT.** Inbound needs a public IP on the NIC or a load balancer. **Outbound** access matters for package installs and external APIs. Azure is moving away from implicit default outbound access for new deployments, so use a **NAT gateway** (a stable shared outbound IP, useful when partners allowlist you) or load balancer outbound rules, and confirm current behaviour.

**Load balancing (conceptual).**

| Service | Layer and scope | Use for |
|---|---|---|
| **Azure Load Balancer** | L4 (TCP/UDP), regional | distributing TCP to VMs with health probes, no HTTP awareness |
| **Application Gateway** | L7, regional, optional WAF | host and URL routing, TLS termination, WAF |
| **Front Door** | L7, global edge, CDN and WAF capabilities | global entry, edge caching, multi-region failover |
| **Traffic Manager** | DNS-based, global | steering clients by DNS answer; no traffic passes through it |

VMs needing HA can use Load Balancer plus nginx on each. HTTP routing, WAF and central TLS suggest Application Gateway. Global audiences suggest Front Door. A layer 7 service in front of nginx adds an `X-Forwarded-For` hop (chapter 4), needs a real health endpoint and aligned idle timeouts (chapter 10). Product lines change, so verify tiers.

**DNS.** **Azure DNS** hosts zones, or use any provider. Point `api.example.com` at the public IP or load balancer. **Private DNS zones** resolve private-endpoint names (a database hostname to a private IP, chapter 18). Lower the **TTL** before a cutover (chapter 20).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| NSG rules | Portal effective security rules, `az network nsg rule create/list` | Explain why a packet is blocked |
| Reachability | `nc -zv host port`, `curl -v`, `ss -tulpn` | Locate the blocking hop |
| Packets and DNS | `tcpdump -i eth0 -nn port 443`, `dig +short api.example.com` | Prove packets arrive; verify records and TTL |

### Common mistakes

- SSH or database ports open to the internet.
- Forgetting a lower-numbered deny overrides a later allow.
- Opening the NSG but not the host firewall, or the reverse; high DNS TTL before a cutover.

### Self-check

- Which NSG rule wins, priority 100 or 200?
- State layer and scope of Load Balancer, Application Gateway and Front Door.

---

## 17. Identity and access

### Understand

**Microsoft Entra ID** (formerly Azure Active Directory) is the identity provider: users, groups, service principals and managed identities, with MFA and conditional access.

**Azure RBAC.** A **role assignment** combines a **principal** (user, group, service principal, managed identity), a **role definition** (permitted actions) and a **scope** (management group, subscription, resource group, resource). Built-in roles include Owner, Contributor and Reader, plus many specific ones. Grant **least privilege** at the narrowest working scope, to **groups** rather than individuals. **Management plane** (create and configure) differs from **data plane** (read a secret or blob): Contributor on a vault does not let you read its secrets.

**Managed identities.** An identity for a resource, created and rotated by the platform, so the app holds **no secret**. **System-assigned** is tied to one resource. **User-assigned** is a separate resource attachable to several VMs and surviving their replacement, which suits blue-green migration. Flow: code requests a token from the instance metadata endpoint, then calls the service (Key Vault, Storage, PostgreSQL with Entra authentication). Azure SDKs wrap this in `DefaultAzureCredential`, using your developer login locally and the managed identity in Azure. Authorise it with a role assignment on the target resource (shapes only, confirm with `--help`):

```
az vm identity assign     -g <rg> -n <vm> --identities <user-assigned-identity-id>
az role assignment create --assignee <principal-id> --role "<role-name>" --scope <resource-id>
```

**Key Vault.** A managed store for **secrets, keys and certificates** with access control, audit logs, versioning, soft delete and purge protection. Access by Azure RBAC (preferred) or legacy access policies, plus network restrictions. Pattern: the VM identity gets a read-secrets data-plane role, the app fetches secrets at startup and caches them in memory, and nothing sits in the repo or ecosystem file. Never write fetched secrets to world-readable files or logs. Long-lived keys leak through repos, logs and laptops, are hard to rotate and leave no usage audit. Managed identity removes the credential for Azure-to-Azure access, while non-Azure systems still need secrets, and Key Vault is where they live.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Role inspection | Portal "Access control (IAM)", `az role assignment list --assignee <id> --all` | Explain why access was granted or denied |
| Managed identity | SDK `DefaultAzureCredential` | Call a service from the VM with no stored secret |
| Auditing | Activity log, Key Vault diagnostic logs | See who changed or read what |

### Common mistakes

- Owner or Contributor at subscription level "to make it work".
- Mixing up management-plane and data-plane roles; not allowing minutes for role propagation.
- System-assigned identity on a VM you will replace: the new VM gets a new principal needing roles again.

### Self-check

- Explain principal, role definition and scope.
- How does a VM app authenticate to Key Vault without a password?

---

## 18. Data services

### Understand

### PostgreSQL flexible server

A managed PostgreSQL service: the platform runs the host, patches, takes backups and offers HA. You get no host shell and no true superuser. Confirm tiers, limits and feature names in Microsoft Learn.

- **Compute and storage.** Choose a compute tier (burstable, general purpose, memory optimised in concept) and storage size. Undersizing shows as high CPU and connection pressure.
- **Networking modes.** **Public access** (public endpoint guarded by firewall rules; "allow Azure services" is broad, avoid it) or **private access** (VNet integration through a delegated subnet, or a private endpoint, with private DNS and no public endpoint). For production prefer private so the database is **never internet reachable**. Some choices are hard to change later, so decide at design time. The app VM must be in the same or a peered VNet, and DNS must resolve the server name to the private IP.
- **TLS and authentication.** TLS is required by default; use `sslmode=require` or `verify-full`. Passwords, or Microsoft Entra authentication where enabled, which pairs with managed identities.
- **Backups and PITR.** Automatic backups with a retention window. PITR restores to any moment in the window **into a new server**. Test a restore before you need it.
- **High availability.** An optional standby, same-zone or zone-redundant. It protects against host or zone failure, **not bad data** (a dropped table replicates). Failover interrupts connections briefly, so apps need reconnect logic.
- **Parameters and connections.** Settings (`max_connections`, `work_mem`, `log_min_duration_statement`) change through the service's parameter interface, some needing restart, some fixed. Extensions must be **allow-listed** (server parameter `azure.extensions`) before `CREATE EXTENSION`. Each connection costs memory, so budget `instances x pool size` below `max_connections`, and use a pooler (PgBouncer; a built-in option exists, confirm) when many processes connect (`09_PostgreSQL_and_Drizzle.md`).

**Migration from a VM-hosted PostgreSQL** (logical dump and restore, for databases that tolerate a window), run from a VM in the same region and VNet with a `pg_dump` client at least as new as the source:

```bash
pg_dump -h old-host -U app -d appdb -Fc -f appdb.dump            # after quiescing writes
pg_restore -h <server>.postgres.database.azure.com -U <admin> -d appdb \
           --no-owner --no-privileges -j 4 appdb.dump             # roles and extensions prepared first
psql -h <server>... -d appdb -c 'ANALYZE;'
```

**Verify, do not assume.** Generate exact per-table counts on both sides and diff them:

```sql
SELECT format('SELECT %L AS t, count(*) FROM %I.%I', relname, schemaname, relname)
FROM pg_stat_user_tables ORDER BY 1 \gexec
```

`n_live_tup` is only an estimate. Also compare `max(id)`, `max(updated_at)`, sums of key numeric columns, sequence values, indexes, constraints and extensions, and run integration tests against the new database before cutover. For large databases needing near-zero downtime use logical replication or Microsoft's migration tooling and cut over when lag reaches zero; consult current documentation.

### Blob Storage, SAS tokens and managed identity

**Blob Storage** holds objects in containers within a storage account. Use it for uploads, exports, backups and static assets rather than VM disks. **Account keys** are full-control shared secrets, so avoid them in applications. A **Shared Access Signature (SAS)** is a signed URL with limited permissions on a resource for a limited time, optionally IP-restricted. A **user delegation SAS** is signed with Entra credentials and is the recommended kind. Use SAS to give a **browser or client** a short-lived direct upload or download URL so large files bypass your VM. SAS URLs are bearer tokens: keep them short-lived, narrow and out of logs. **Managed identity with RBAC** (a blob data role) is right for **server-to-storage** calls: no secret, auditable, revoked by removing the role. Rule: the backend uses managed identity and mints user delegation SAS for clients. Keep containers private and use lifecycle rules, soft delete and versioning.

### Azure Cache for Redis (concepts)

A managed Redis for caching, sessions, rate limiting and pub/sub (`11_Redis.md`). Tiers differ in memory, persistence, clustering and replication. Access is over TLS with an access key or Entra authentication, and networking is public with a firewall or private through a private endpoint. Microsoft has been evolving its Redis offerings and announced retirement paths for some tiers with a successor service, so **check the current product line and lifecycle announcements before choosing**. Stable design points: treat the cache as **disposable** (the app must still work, slower, when empty), set TTLs and an eviction policy, handle reconnects with a retry policy, co-locate it in the app's region, and never make it the only copy of data.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Connectivity | `nc -zv <server> 5432`, `dig <server>`, `psql "host=... sslmode=require"` | Prove private DNS and port reachability |
| Dump, restore, verify | `pg_dump -Fc`, `pg_restore -j --no-owner`, exact `count(*)` | Migrate and prove equality before cutover |
| Backups and storage | PITR restore to a new server, SDK with `DefaultAzureCredential`, user delegation SAS | Time a restore drill; choose the right storage access |

### Common mistakes

- Public database access with a wide firewall range.
- Never testing a restore; assuming HA protects against corruption or a bad migration.
- Pool size times workers above `max_connections`; comparing only `n_live_tup` estimates.
- Account keys or long-lived SAS in code or logs.

### Self-check

- Public versus private access for PostgreSQL flexible server: which for production and why?
- What does PITR restore into, and what does HA not protect against?
- When managed identity for Blob, and when SAS? How do you prove a restored database matches the source?

---

## 19. Monitoring, cost control and security baseline

### Understand

**Azure Monitor** covers **metrics** (numeric time series, cheap, near real time) and **logs** (events, stored and queried).

- **Platform metrics** are automatic: VM CPU, network, disk operations, burstable credits, database CPU, storage, connections. **Guest memory and in-guest disk space are not visible to the hypervisor**, so they need the **Azure Monitor Agent** with a **data collection rule** (older agents are being retired; confirm).
- **Log Analytics workspace** is the log store, queried with **KQL**. Send syslog, performance counters, NSG flow logs, Key Vault and database diagnostics and the **Activity Log** (control-plane events: "who changed the NSG?"). nginx, pm2 and systemd logs arrive through agent collection or an app library. Ingestion and retention drive cost, so collect deliberately.
- **Alerts**: a rule evaluates a metric or log query and fires an **action group** (email, SMS, webhook, chat). Start with VM heartbeat missing, sustained CPU, low available memory, disk usage, 5xx rate from nginx logs, certificate days to expiry, pm2 restart growth, database CPU, storage and connections, and budget thresholds. Alert on **user-felt symptoms** (error rate, latency) first and saturation second. Every alert needs an owner and a runbook.
- **Boot diagnostics and serial console** help with unreachable VMs. **External availability checks** of `https://api.example.com/health` catch DNS, TLS and NSG faults internal checks miss. An alert is a prompt to run the one-minute checklist in `03_Linux.md`, chapter 6.

**Cost control.** Azure bills for what is **allocated**, not used. Confirm current pricing models in Azure pricing documentation.

- **Right-size** from weeks of utilisation (Azure Advisor helps). **Deallocate non-production** outside working hours and **delete orphans** (unattached disks, unused public IPs, old snapshots, idle load balancers).
- **Reserved instances** and **savings plans** trade a one- or three-year commitment for a discount on steady workloads: commit only to the baseline, after right-sizing. **Spot VMs** can be evicted, so they suit batch work, not your API.
- Egress out of a region or Azure is billed, so keep chatty components co-located. A **budget** at subscription or resource-group scope with threshold alerts notifies but does not stop spending. **Tag** resources so cost analysis can attribute spend. A managed database costs more in compute but includes backups, patching, HA and your time.

**Security baseline.**

- **Network:** NSG default deny, only 80 and 443 public (or only the load balancer), SSH from a known range or Bastion, no public database or Redis, app ports on loopback or private only, private endpoints where available.
- **Identity:** MFA for humans, least-privilege RBAC on groups, managed identities over keys, secrets in Key Vault, no shared accounts, periodic role review.
- **Host:** key-only SSH, `PermitRootLogin no`, non-root service users, `ufw` as a second layer, `fail2ban` (`03_Linux.md`, chapter 12). Unattended security updates, a reboot window, `needrestart`, and patching of nginx, Node, Python and OpenSSL, with dependency audits (`npm audit`, `pip-audit`).
- **Microsoft Defender for Cloud** (concept): posture management and threat protection. It scores configuration against recommendations (open management ports, missing patches, public storage) and, on paid plans, detects threats on servers and databases. Treat recommendations as a prioritised backlog and confirm plan names in current docs.
- **Data and detection:** TLS everywhere including to the database, encryption at rest (platform-managed by default), private blob containers, soft delete, tested backups kept outside the app's credentials. Keep the Activity Log, `auth.log`, nginx logs and Key Vault audit logs centrally, and alert on failed-login bursts, new role assignments and new public IPs.
- **Change control:** create infrastructure from reviewed scripts or templates, and add rate limits, body limits, security headers and a WAF for internet-facing APIs when risk justifies it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Metrics, logs, alerts | Azure Monitor metrics explorer, Log Analytics, KQL (`where`, `summarize`, `project`), alert rules, action groups | Chart saturation; query 5xx per five minutes; create an alert with owner and threshold |
| Cost | Cost Management, cost analysis, tag filters, Azure Advisor, budgets, `az vm deallocate` | Explain last month's bill by app; find idle resources |
| Security audit | `ss -tulpn`, `nmap` from outside, NSG effective rules, `lynis`, Defender recommendations | Prove only intended ports are reachable |

### Common mistakes

- Assuming memory and disk-space metrics exist without an agent; no alert for certificate expiry or pm2 restarts.
- Believing a stopped (not deallocated) VM is free; buying a reservation before right-sizing; no tags or budget.
- A public database "for convenience" that stays forever; backups in the same blast radius as production.

### Self-check

- Which VM metrics need an in-guest agent and why, and what do you alert on first?
- Why does deallocating save compute cost, and what still bills?
- List the top exposures on a fresh internet-facing VM and the control for each.

---

## 20. Reference deployment walkthrough, end to end

Goal: a Node API (and optionally a Python service) on an Ubuntu LTS VM, behind nginx with TLS, talking to managed PostgreSQL privately, observable and recoverable, then migrated to a new VM with no downtime. Names are placeholders. Confirm exact commands and portal steps in current documentation.

**Step 1. Plan.** Decide region, subscription, resource group (`rg-app-prod`), VNet range, subnets, tags, DNS name and supervisor (pm2 cluster for the Node API, systemd plus gunicorn for Python). Write the port map and the secrets list.

**Step 2. Provision network and VM.** Create the resource group, VNet and subnets. Create an NSG allowing 443 and 80 from the internet and 22 only from your admin range (or use Bastion), relying on default deny, associated with the app subnet. Create the VM from an Ubuntu LTS image with **SSH key authentication only**, a Standard static public IP (or a private IP behind a load balancer), **boot diagnostics enabled**, tags, optional cloud-init and a **user-assigned managed identity**. Create the A record for `api.example.com` with a low TTL.

**Step 3. Harden.** Log in with your key. Create a `deploy` user, set `PasswordAuthentication no`, `PermitRootLogin no`, `AllowUsers deploy`, and test in a **second session** before closing the first (`03_Linux.md`, chapter 12). Enable `ufw` (22 from your range, 80, 443), `unattended-upgrades`, time sync and `fail2ban`. Create a non-login `app` service user.

**Step 4. Install runtime and app.** Install Node LTS by an official pinned method and Python in a `venv`. Create `/opt/app/releases/<id>` with a `current` symlink owned by `app`. Install from the lockfile (`npm ci`), build, and supply configuration through a `600` env file or Key Vault. The app **listens on `127.0.0.1:3000`**, exposes `/health` (cheap DB check) and shuts down gracefully (chapter 12).

**Step 5. Supervise.** Node: the ecosystem file from chapter 11 (cluster, 2 instances, `wait_ready`, `kill_timeout`, `max_memory_restart`, backoff, `max_restarts`), then `pm2 start ... --env production`, `pm2 save`, `pm2 startup`, `pm2-logrotate`. Python: the systemd unit from chapter 13. Check `curl http://127.0.0.1:3000/health`, then reboot once and confirm everything returns.

**Step 6. nginx and TLS.** Install nginx. Create the site file with the HTTP redirect, TLS server block, forwarded headers, timeouts, `client_max_body_size`, the `timed` log format, gzip, API rate limit, catch-all default server and `server_tokens off`. Run certbot with port 80 open, `certbot renew --dry-run`, `nginx -t`, reload. Set the app's `trust proxy` to one hop. Verify:

```
curl -I http://api.example.com            # 301 to https
curl -v https://api.example.com/health    # 200, valid chain, HSTS header
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null | openssl x509 -noout -dates
```

Confirm port 3000 is **not** reachable from outside and `ss -tulpn` shows `127.0.0.1:3000`.

**Step 7. Managed PostgreSQL, private.** Create a flexible server with **private access**, a private DNS zone linked to the VNet, TLS required, defensible backup retention, HA if the target justifies it and a maintenance window. Create the app database and a **least-privilege role** (not the admin). From the VM: `dig <server-name>` returns a **private IP**, `nc -zv <server> 5432` succeeds, and `psql "host=<server> dbname=appdb user=app sslmode=require"` connects. Migrate with `pg_dump -Fc` and `pg_restore --no-owner --no-privileges -j`, verify counts (chapter 18), point the app at the new host and reload. Record the connection budget.

**Step 8. Logs and alerts.** Install the Azure Monitor Agent with a data collection rule for syslog and performance counters including memory and disk, and collect nginx logs. Create the alerts listed in chapter 19, an external probe of `/health` and a budget. Document where each log lives and the first three incident commands.

**Step 9. Zero-downtime VM migration (blue-green).** Reason: replace the VM (size, image, OS upgrade, region) with no maintenance window. **Blue** is current, **green** is new.

1. **Build green identically**: same scripts or cloud-init, same user-assigned identity (roles already apply), NSG rules, nginx config and app version. Keep it out of rotation.
2. **Prepare data.** A managed shared database needs no move: green connects to it. A VM-local database is migrated first (chapter 18), with a final sync under a short write freeze or logical replication until lag is zero. Verify **row counts and checksums** before any traffic moves.
3. **Test green** with `curl --resolve api.example.com:443:<green-ip> https://api.example.com/health` and a smoke suite. For TLS on green use DNS-01, copy the certificate securely and set up renewal, or terminate TLS at a load balancer or Application Gateway.
4. **Lower DNS TTL** (for example 60 seconds) at least one old-TTL period **before** cutover.
5. **Prevent duplicate side effects.** Anything scheduled or consuming a queue must run in **exactly one** place. Cron, `node-cron` and systemd timers: disabled on green until cutover and on blue right after, or guarded by a **distributed lock** (database advisory lock, Redis lock with TTL). Queue consumers: stop and drain blue's as green's start, or run competing consumers with **idempotent handlers**. Never run a second scheduler. Webhooks, email, payments and other non-idempotent jobs need idempotency keys.
6. **Cut over.** Switch the **DNS A record** to green (clients move as caches expire, so blue keeps serving while traffic drains), or better, place both behind **Azure Load Balancer or Application Gateway**, add green, wait for healthy probes, then remove blue with connection draining. A pool switch is faster and more reversible than DNS. Watch 5xx rate, latency and app errors.
7. **Hold blue** for a bake period. Rollback means moving the pool or DNS back, which works only if data changes remain readable by blue, so keep migrations backward compatible. Deallocating blue saves compute cost but slows rollback.
8. **Decommission** blue after the bake period: confirm no traffic in nginx logs, take a final snapshot, remove identity-specific role assignments, then delete the VM, disks, NSG rules and public IP. Raise the DNS TTL again and record lessons.

---

## 21. Deployment checklist, rollback template, 502 incident playbook

### A. Deployment checklist

```
Before
[ ] Release built from a tagged commit with a lockfile; tests pass; /health reflects real readiness
[ ] Env var and secret changes reviewed; migrations backward compatible and tested on a copy
[ ] Recent backup or PITR window confirmed; restore procedure known
[ ] nginx changes pass `nginx -t`; certificate not near expiry; NSG, DNS and TTL as intended
[ ] Disk, memory headroom and DB connection budget checked
[ ] Previous release kept on disk; rollback command prepared
[ ] pm2 reload requirements met (cluster, >= 2 instances, signals handled, wait_ready, kill_timeout)
After
[ ] pm2 ls: all online, restart count unchanged, uptime growing
[ ] curl -v https://api.example.com/health returns 200 through nginx
[ ] No new upstream errors in error.log; 5xx rate and p95 latency flat
[ ] Schedulers and queue consumers running once, not twice; key business flow smoke-tested
```

### B. Rollback plan template

```
Service / release: ________   Previous good release: ________
Owner / approver:  ________   Decision deadline:      ________
Triggers (any):    5xx > __% for __ min | p95 > __ ms | health fails | data error
Code:              point `current` symlink at <previous>; pm2 reload ... --update-env; verify /health and version
Config:            restore previous env file and nginx config from VCS; nginx -t; reload
Database:          migrations backward compatible? yes / no
                   if no: restore plan (PITR to new server, ETA __, data-loss window __)
Traffic (blue-green): move LB pool or DNS back to <previous>; TTL = __ s
Jobs and queues:   which schedulers and consumers to stop or start
Verification:      commands and expected output
Communication:     who is told, where, when
Post-incident:     timeline, cause, prevention items
```

Rehearse the rollback on staging. An untested rollback is a guess.

### C. Incident playbook: 502 Bad Gateway after a deploy

Principle: observe before changing, preserve evidence, change one thing at a time, and prefer rollback over live debugging when users are affected.

1. **Confirm and scope.** `curl -v https://api.example.com/health` and a real endpoint. All requests or some? If impact is large and a deploy just happened, **roll back first** (section B), then investigate.
2. **Does traffic reach nginx?** If the response carries your nginx `Server` header, yes. If not, check DNS, NSG and the load balancer or Application Gateway probe status.
3. **Read `error.log`** at the failure time. `Connection refused`: the app is not listening on that address and port. `upstream prematurely closed connection`: crash or reset mid-request. `too big header`: raise `proxy_buffer_size`. `no live upstreams`: all backends failed. `upstream timed out`: a 504 pattern, slow or blocked app.
4. **Test the app on the VM:** `curl -v http://127.0.0.1:3000/health`. Fails: the app. Works: nginx routing (wrong port, `proxy_pass` slash, wrong upstream) or a different app version bound.
5. **Check the supervisor.** `pm2 ls` or `systemctl status`: **restart count** and uptime. `pm2 logs --lines 200` or `journalctl -u <svc> --since "15 min ago"`. Common post-deploy causes: missing or renamed environment variable, missing build artifact, changed port, startup exception, migration not applied, secret fetch failing (identity role missing on the new VM or not yet propagated), wrong database host or credentials.
6. **Check resources and dependencies.** `ss -tulpn`, `free -h`, `df -h`, `dmesg -T | grep -i killed` (a reload briefly doubles worker memory and can OOM a small VM), `nc -zv <db-host> 5432`, `dig <db-host>` (private IP?), connection count versus `max_connections`. Sporadic 502s suggest an app keep-alive shorter than nginx's, workers exiting without draining on reload, or `kill_timeout` too short.
7. **Mitigate.** Roll back, or fix forward only when the cause is clear and the fix small and verified. Do not restart repeatedly or just raise timeouts.
8. **Verify** with the "After" checklist, watch 15 to 30 minutes, then write a blameless timeline: symptom, evidence, cause, fix, prevention (readiness check, startup config validation, `/health` deploy gate).

---

## 22. Practice labs

Use a throwaway Azure subscription or any Linux VM. Delete resource groups when finished to avoid cost.

1. **Location and slash puzzle.** Write the five-location example from chapter 3, predict the handler for ten URLs, prove it with an `X-Loc` header, break it with a regex and fix it with `^~`. Run an app that echoes the received path and record it for the four `proxy_pass` slash combinations.
2. **Trust proxy.** Send a forged `X-Forwarded-For`. Show the app logging the forged value, then the real client with `trust proxy` set to one hop.
3. **Keepalive 502.** Set the app keep-alive below nginx's, generate load with idle gaps, reproduce "upstream prematurely closed", then fix it.
4. **Streaming and WebSocket.** Show SSE arriving in one burst with buffering on and incrementally with `proxy_buffering off`. Proxy a WebSocket echo server with and without Upgrade headers.
5. **Rate limit and error catalogue.** Configure `limit_req` with `burst` and `nodelay` and show 429s. Cause 502, 504, 413 and 499 deliberately and record the log lines for each.
6. **TLS and renewal.** Issue a Let's Encrypt certificate, run `certbot renew --dry-run`, close port 80 in the NSG to see renewal fail, restore it.
7. **pm2 cluster.** Run two instances, show in-memory counters diverging, move state to Redis, and show `node-cron` running twice.
8. **Zero-downtime reload.** Run a `curl` loop during `pm2 restart` (count failures), then `pm2 reload` with `wait_ready` and signal handling (zero failures).
9. **Crash loop and boot.** Make the app exit 2 seconds after start, add `min_uptime`, `max_restarts` and backoff, then run `pm2 startup` and `pm2 save`, reboot, and repeat without `pm2 save`.
10. **Python under systemd.** Run a FastAPI or Flask app under gunicorn with a unit file behind nginx and do a graceful `systemctl reload`.
11. **Azure foundations.** With `az`, create a resource group, VNet, NSG with restricted SSH and a key-auth Ubuntu VM with boot diagnostics. Show other ports closed. Deallocate and compare billing states.
12. **Managed identity and Key Vault.** Store a secret, assign a user-assigned identity, grant a read-secret role, fetch it with the SDK with no stored credential, remove the role and watch it fail.
13. **Private PostgreSQL, migration, PITR.** Create a flexible server with private access, verify `dig` returns a private IP, migrate a sample database with `pg_dump -Fc` and `pg_restore`, diff exact per-table counts, then delete rows and recover them with a PITR restore to a new server.
14. **Blue-green cutover.** Build a second VM, switch via load balancer or DNS, guard a scheduled job with a lock so it runs once, then roll back and forward.
15. **Incident drill.** Have a partner break a deploy (env var removed, port changed, `proxy_pass` slash, OOM, DB firewall) and diagnose with the chapter 21 playbook only. Write the timeline.

---

## 23. Interview questions

Every question is followed by a concise model answer.

### Junior

**1. What is a reverse proxy and why put nginx in front of a Node or Python app?**
It accepts client connections and forwards them to backends. nginx terminates TLS, serves static files, buffers slow clients, enforces limits, routes several apps by host or path and keeps the app port private on loopback.

**2. What is the difference between `nginx -s reload` and a restart?**
Reload is graceful: new workers start with the new config and old ones finish their requests, and an invalid config leaves the old one running. Restart drops connections. Run `nginx -t` first.

**3. Which headers does a proxy typically set and why?**
`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, so the app knows the original host, client address and scheme instead of `127.0.0.1` over HTTP.

**4. What is pm2, and fork versus cluster mode?**
A process manager that restarts crashes, starts at boot (`pm2 startup` and `pm2 save`), captures logs and offers rolling reloads. Fork runs one process, any language. Cluster (Node only) forks N workers sharing one port via Node's `cluster` module, to use multiple cores.

**5. What do 502 and 504 mean from nginx?**
502: no valid upstream response (refused, closed early). 504: the upstream did not answer within the proxy timeout.

**6. What does an NSG do, and why use SSH keys on a VM?**
An NSG is a stateful filter on a subnet or NIC with prioritised rules; inbound is denied by default and the lowest priority number matches first. Keys resist brute force and password login can be disabled.

**7. Stopping versus deallocating an Azure VM?**
Stopping inside the OS keeps compute allocated and billed. Deallocating releases compute and stops compute charges, while disks and static IPs still bill.

**8. Why must port 80 stay open for Let's Encrypt HTTP-01?**
Validation arrives on port 80 and repeats at each renewal. Blocking it makes renewal fail until the certificate expires.

### Mid-level

**9. How does nginx choose a location?**
Exact `=` wins. Otherwise the longest prefix is remembered, and if it has `^~` it wins. Otherwise regex locations are tested in file order, first match wins. If none match, the longest prefix is used.

**10. What does the trailing slash in `proxy_pass` do?**
No URI part forwards the original URI. A URI part (even `/`) replaces the matched location prefix, so `location /api/` with `proxy_pass http://u/` sends `/api/x` as `/x`.

**11. Why does an app need `trust proxy`, and what is the risk of getting it wrong?**
It must read client IP and scheme from forwarded headers at the right hop. Too trusting allows spoofing and bypassing IP controls. Too little makes all clients look like the proxy and breaks HTTPS detection.

**12. Intermittent 502s on an otherwise healthy app. What do you suspect?**
Keepalive reuse of a connection the app already closed (app keep-alive shorter than nginx's), workers exiting without draining on reload, OOM kills, or a crash loop. Correlate `error.log` with app logs, restart counts and `dmesg`.

**13. How do you stream SSE through nginx, and how do you proxy WebSockets?**
SSE: HTTP/1.1, empty `Connection` header, `proxy_buffering off` (or `X-Accel-Buffering: no`) and a long `proxy_read_timeout`. WebSockets: HTTP/1.1, forward `Upgrade` and `Connection` via a `map` on `$http_upgrade`, raise `proxy_read_timeout`, use heartbeats, expect `101`.

**14. What are the limits of `limit_req` behind a CDN?**
All requests share the CDN's IPs, so everyone shares a bucket. Use `realip` with trusted ranges or limit at the edge. State is per nginx instance and per-IP is not per-user.

**15. What is required for a zero-downtime `pm2 reload`?**
Cluster mode with at least two instances, a readiness signal (`wait_ready` plus `process.send('ready')`), graceful shutdown on `SIGINT` and `SIGTERM`, `kill_timeout` above the drain time, and backward-compatible database changes.

**16. Why is `max_memory_restart` not a fix for a leak?**
It bounds damage by restarting at a threshold, hides the leak and can drop work. Find the leak with heap snapshots and alert on restarts.

**17. systemd with gunicorn or pm2 for a Python service?**
systemd with gunicorn. gunicorn manages workers and graceful reloads, systemd supplies cgroup limits, ordering and journald, and pm2 cluster mode does not apply to Python.

**18. Compare Azure Load Balancer, Application Gateway and Front Door.**
Load Balancer is L4 and regional. Application Gateway is L7 and regional with TLS termination, routing and optional WAF. Front Door is a global L7 entry with edge, CDN and failover capabilities.

**19. How does a VM authenticate to Key Vault without a secret?**
A managed identity is attached to the VM and granted a data-plane role. The SDK gets a token from the instance metadata endpoint and presents it to Key Vault. No credential is stored.

**20. Public versus private access for PostgreSQL flexible server?**
Public exposes an endpoint guarded by firewall rules. Private places the server in your VNet with private DNS. Choose private for production, and decide early since some choices are hard to change.

**21. How do you migrate a database from a VM to the managed server and verify it?**
Quiesce writes, `pg_dump -Fc`, create roles and allow-list extensions, `pg_restore --no-owner --no-privileges -j`, `ANALYZE`, then compare exact per-table counts, aggregates and sequences and run application tests. Statistics views give estimates only.

**22. SAS or managed identity for Blob access?**
Managed identity with RBAC for your backend. A short-lived, narrowly scoped user delegation SAS for giving a client direct access. Never log SAS URLs.

### Senior

**23. Design a Node API deployment on Azure VMs with HA and zero-downtime releases.**
At least two VMs across zones behind a load balancer or Application Gateway probing a real `/health`. nginx on each, pm2 cluster or systemd for the app, managed PostgreSQL with HA over a private path, managed identity and Key Vault, a stateless app with Redis for shared state, rolling releases (drain, deploy, verify, re-add), expand-contract migrations, monitoring, alerts and a tested rollback.

**24. Walk through a blue-green VM migration and its pitfalls.**
Build green identically, verify data, test via `--resolve`, lower TTL early, ensure schedulers and consumers run exactly once (locks, idempotent handlers), shift via pool or DNS, watch errors, hold blue for a bake period, decommission. Pitfalls: duplicate cron, split writes, DNS caches, TLS on green, migrations that block rollback.

**25. 502s for 30 seconds on every deploy. Diagnose.**
Workers likely die before draining or before new ones are ready: one instance, no `wait_ready`, no signal handler, short `kill_timeout`, or `restart` rather than `reload`. Reproduce with a request loop, read pm2 and nginx logs during reload, fix and verify zero failures.

**26. How do you align timeouts across load balancer, nginx, app and database?**
Make inner timeouts shorter than outer ones so the innermost layer fails first with a meaningful error, and keep keep-alive idle timeouts increasing inward (app longer than nginx, nginx longer than the outer layer's reuse window). Mismatches show as 499, 502 or 504 at different layers.

**27. How do you reduce Azure spend without hurting reliability, and how do you remove long-lived secrets?**
Right-size from measurements, deallocate non-production, delete orphans, tag, commit reservations only for the right-sized baseline, cap log ingestion and set budgets. For secrets: managed identities, user delegation SAS for clients, Key Vault with RBAC and audit logs, nothing in the repo or ecosystem file, and secret scanning in CI.

**28. How do you ship a database change with no downtime and a safe rollback?**
Expand then contract: add compatible structures, deploy code that handles both, backfill in batches, switch reads and writes, remove the old structure later. Keep each step reversible, confirm a backup or PITR window, and know the restore-to-new-server procedure.

**29. Rate limiting for a public API behind Front Door and nginx: what goes where?**
Coarse abuse and DDoS protection at the edge (WAF, rate rules), per-IP burst protection at nginx using the true client IP from trusted headers, and per-user or per-key quotas in the app with Redis shared state. No single layer suffices, and client-IP handling must be consistent across layers.

**30. How do you make sure certificate renewal does not cause an outage in 90 days, and when do you outgrow one VM with nginx and pm2?**
Test with `certbot renew --dry-run`, confirm a timer and a deploy hook that reloads nginx, keep port 80 or the DNS-01 path available, and alert on days to expiry from an external check. You outgrow the setup when you need consistent multi-VM environments, orchestrated rollouts, autoscaling or immutable infrastructure, or when manual patching becomes the burden.

---

## 24. References

Primary sources, preferred over blog posts. Verify each URL before depending on it, since pages move.

**nginx and TLS**
- nginx documentation (modules `ngx_http_proxy_module`, `ngx_http_upstream_module`, `ngx_http_core_module`, `ngx_http_limit_req_module`, `ngx_http_ssl_module`, `ngx_http_realip_module`, variables index): https://nginx.org/en/docs/
- nginx WebSocket proxying: https://nginx.org/en/docs/http/websocket.html
- Mozilla SSL Configuration Generator: https://ssl-config.mozilla.org/
- Certbot documentation: https://eff-certbot.readthedocs.io/
- Let's Encrypt documentation: https://letsencrypt.org/docs/
- HAProxy documentation: https://docs.haproxy.org/
- Caddy documentation: https://caddyserver.com/docs/

**pm2, runtimes and supervisors**
- PM2 documentation (ecosystem file, cluster mode, restart strategies, graceful shutdown, startup): https://pm2.keymetrics.io/docs/usage/quick-start/
- Node.js `cluster`: https://nodejs.org/api/cluster.html
- Node.js `http` (`keepAliveTimeout`, `close`): https://nodejs.org/api/http.html
- Express behind proxies: https://expressjs.com/en/guide/behind-proxies.html
- Fastify server options (`trustProxy`): https://fastify.dev/docs/latest/Reference/Server/
- Gunicorn documentation: https://docs.gunicorn.org/
- systemd manual pages: https://www.freedesktop.org/software/systemd/man/

**Azure** (Microsoft Learn: https://learn.microsoft.com/azure/)
- Resource organisation, naming and tagging; regions and availability zones
- Virtual machines (Linux quickstarts, sizes, disks, cloud-init, boot diagnostics)
- Virtual network, network security groups, application security groups, Azure DNS and private DNS
- Load balancing options: Load Balancer, Application Gateway, Front Door, Traffic Manager
- Microsoft Entra ID, Azure RBAC, managed identities, Azure Key Vault
- Azure Database for PostgreSQL flexible server (networking, backup and restore, high availability, server parameters, migration)
- Azure Blob Storage authorisation, SAS and user delegation SAS; Azure Cache for Redis and current lifecycle pages
- Azure Monitor, Log Analytics, Azure Monitor Agent, alerts; Cost Management, budgets, Azure Advisor; Microsoft Defender for Cloud
- Azure CLI reference: https://learn.microsoft.com/cli/azure/

**PostgreSQL**
- `pg_dump`, `pg_restore`, logical replication, statistics views: https://www.postgresql.org/docs/current/

**Companion material in this repository**
- `03_Linux.md`: systemd, signals, networking, TLS, SSH, logs.
- `CloudDevOps/`: AWS-focused cloud and delivery material that complements this Azure-focused guide.
