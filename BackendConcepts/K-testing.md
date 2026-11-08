# Section K — Testing

> Reference outline.

## K1 — The test pyramid

**Diagram**
```
        /\        E2E   (few, slow, whole flow)
       /  \
      /----\      Integration (some, real DB/Redis)
     /------\
    /--------\    Unit (many, fast, isolated)
```
**In plain words.** Lots of cheap quick checks at the bottom, a few expensive full rehearsals at
the top.

- Many **unit** tests (fast, isolated), fewer **integration** tests (real DB/queue), few **end-to-end**
  (whole flow). Fast feedback at the bottom, confidence at the top.
- One line: mostly unit, some integration, a few E2E.

## K2 — Unit testing

**Diagram**
```
 test: service.createOrder(input)
   repo = fake/mock, notifier = fake/mock
   assert result + calls + edge cases (empty, negative, duplicate)
 works because services have no req/res (B6)
```
**In plain words.** Test one machine part on the bench, with pretend parts around it.

- Test one unit (a service function) with dependencies mocked. Assert behavior and edge cases. Layering
  (B6) makes services unit-testable because they're HTTP-free.
- One line: unit-test HTTP-free service logic with mocked deps.

## K3 — Integration testing

**Diagram**
```
 test -> service -> REAL Postgres (container) + REAL Redis
 catches: wrong query, broken migration, transaction bugs, wiring
```
**In plain words.** Now assemble the parts and check they actually fit together.

- Test across real components (service + real DB/Redis, often via testcontainers). Catches wiring,
  queries, migrations, transactions.
- One line: integration-test against real DB/Redis to catch wiring and query bugs.

## K4 — API / contract testing

**Diagram**
```
 POST /orders  -> expect 201 + {success:true, data:{...DTO shape}}
 bad input     -> expect 400 + {success:false, error:{code:"VALIDATION"}}
 contract test: producer and consumer agree on the shape
```
**In plain words.** Test the API like a client would, from the outside.

- Hit endpoints and assert status + envelope + DTO shape (B7). Contract tests verify producer/consumer
  agreement between services.
- One line: test the API surface (status/envelope/shape) and contracts between services.

## K5 — Mocking external services

**Diagram**
```
 code -> fake KYC provider/email provider/SMS provider (returns scripted replies)
 script: success, 500, timeout, slow -> prove retries/breaker/state machine work
```
**In plain words.** Use a stunt double for the third party. You can make it fail on purpose.

- Mock a KYC provider/email provider/SMS provider so tests are deterministic and offline; test failure paths
  (timeouts, 5xx) too.
- One line: mock third parties for determinism, and test their failure modes.

## K6 — Coverage & what to test

**Diagram**
```
 coverage 95% of getters  <  coverage of: auth + ownership + validation + money + KYC
 spend effort where a bug hurts most
```
**In plain words.** A coverage number is not safety. Test the dangerous parts hardest.

- Coverage is a guide, not a goal. Prioritize business rules, auth/ownership (C9), validation (D1),
  money/KYC paths.
- One line: chase risk, not a coverage number — cover auth, validation, and money paths hardest.

## K7 — Test data & fixtures

**Diagram**
```
 each test: start clean (rollback or fresh DB) -> seed fixed data
 seed TWO tenants: org A and org B
 -> a missing org_id filter shows org B's data in org A's result -> test fails
```
**In plain words.** Always test with two companies in the database, so a forgotten filter shows up
at once.

- Deterministic seed data, isolated per test (transaction rollback or fresh DB), multi-tenant data so
  org-filter bugs (C10) surface.
- One line: use isolated, deterministic fixtures — include multi-tenant data.
