# Section L — DevOps / Delivery

> Reference outline.

## L1 — CI (continuous integration)

**Diagram**
```
 git push -> npm ci -> lint -> test -> security scan -> build -> green/red
 keep main green
```
**In plain words.** An automatic checker that runs the same checklist on every change.

- On every push: install from lockfile (npm ci), lint, test, **security scan** (npm audit/Snyk, D17),
  build. Fail fast; keep main green.
- One line: CI runs lint + tests + security scan + build on every push.

## L2 — CD (continuous delivery/deployment)

**Diagram**
```
 build ONE image -> staging (tests, gate) -> approve -> prod (same image)
```
**In plain words.** Ship the same sealed box to every stage. Never rebuild it per environment.

- Automated deploys to staging/prod with gates. Reproducible builds; immutable artifacts/images.
- One line: automate deploys with gates and immutable artifacts.

## L3 — Environments & config

**Diagram**
```
 same code + different config:
 dev (.env dev) | staging (env + vault) | prod (env + vault)
 nothing environment-specific in code
```
**In plain words.** One recipe, different ingredients per kitchen.

- Separate dev/staging/prod; config via env vars + secrets manager (D11); never hardcode per-env values.
- One line: separate environments, config via env + secrets manager.

## L4 — Containers & images

**Diagram**
```
 multi-stage: [build stage: tools, devDeps] -> copy only dist -> [slim runtime]
 run as non-root, pin versions, scan (Trivy)
```
**In plain words.** Ship only the finished product, not the whole factory.

- Slim base images, multi-stage builds, non-root user, scan images (D19), pin versions.
- One line: build slim, non-root, scanned, version-pinned images.

## L5 — Deploy strategies

**Diagram**
```
 rolling   : replace instances one by one
 blue-green: v2 stack ready, switch traffic at once, switch back if bad
 canary    : 5% of users on v2 -> 50% -> 100%
```
**In plain words.** Three ways to change the engine of a moving train, each with a different risk.

- **Rolling / blue-green / canary** for zero-downtime; pair with backward-compatible migrations (E4).
- One line: deploy with rolling/blue-green/canary + backward-compatible migrations.

## L6 — Rollback & migrations

**Diagram**
```
 deploy v2 broke? -> roll back to v1 fast
 schema must work for BOTH: add column first, old code ignores it
 EXPAND -> deploy -> MIGRATE data -> CONTRACT later
```
**In plain words.** Never make a change that you cannot undo within minutes.

- Be able to roll back fast; expand→migrate→contract schema changes so old and new code coexist during
  deploy.
- One line: keep rollbacks fast and schema changes backward-compatible.

## L7 — Zero-downtime deploys

**Diagram**
```
 SIGTERM -> readiness=false -> LB stops sending -> finish in-flight
         -> close DB/Redis/workers -> exit
```
**In plain words.** The shop puts up a "closing" sign, serves the people inside, then locks up.

- Health checks (H5), graceful shutdown (drain in-flight requests, close pools), connection draining at
  the LB.
- One line: drain gracefully and use health checks for zero-downtime.
